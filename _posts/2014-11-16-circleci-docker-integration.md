---
layout: post
title: "Using CircleCI with Docker"
comments: true
permalink: "cockroach-docker-circleci-continuous-integration"
---

*I recently spent a weekend switching [Cockroach](https://github.com/cockroachdb/cockroach) to [CircleCI](http://circleci.com) with [Docker](http://docker.io) for our continuous integration needs. Previously, we had a standard build script on [Travis CI](https://travis-ci.org/), but with a variety of dependencies that needed to be compiled and vendored in, the average build took about ten minutes to get to the unit tests.
Now we clock in at about half that but are able to actually orchestrate a [Cockroach](https://github.com/cockroachdb/cockroach) cluster and run acceptance tests. But there are some small kinks that are useful to know about.*

Cockroach (at the time of writing) comes with three images:

* `cockroachdb/cockroach-devbase`: A development base image.
* `cockroachdb/cockroach-dev`: The development image, based on
 cockroach-devbase.
* `cockroachdb/cockroach`: A small, statically linked
  [Cockroach](https://github.com/cockroachdb/cockroach) binary inside
  of a [BusyBox](http://busybox.net).

We want [CircleCI](http://circleci.com) to build those three images and run a bunch of tests.

## Caching Docker images
Unfortunately, while Docker locally caches the result of every build step and will blaze through without doing any actual work until it encounters a step that breaks the cache (for instance because a file from the build context changed), on [CircleCI](http://circleci.com) each build runs on a fresh virtual machine somewhere on their cluster.
This means that unless you're smart about it, Docker will always build everything from the very first step. But the goal was to not have to that - what next?

Luckily, [CircleCI answers this](https://circleci.com/docs/docker) themselves and advises you to use `docker save` to export the built image into a tar file, which you can then tell them to cache:

```
dependencies:
  cache_directories:
    - "~/docker"

  override:
    - if [[ -e ~/docker/image.tar ]]; then \
        docker load -i ~/docker/image.tar; fi
    - ./build/build-docker.sh
    - mkdir -p ~/docker; docker save cockroachdb/cockroach-devbase \
        > ~/docker/image.tar
```

While this works, **it costs time**. For our ~1GB `cockroachdb/cockroach-devbase` image, about a minute to load and a little less to save. Plus, the data stored on the cache has to be copied over the network to our vm every time a test runs - another noticeable fraction of a minute.

So we pay a price here to use Docker, but yes - once we have the base image loaded, building the actual `cockroachdb/cockroach-dev` image is very fast - way less than a minute.

### Git kills the cache
If you've made it that far, you'll be fairly happy, but if you're building containers that get the `.git` directory in their build context, you'll notice that it is **never cached**.

That's because CircleCI gives you (more or less) a fresh repo every time - each file will be new and Docker will invalidate its cache for them, even if they haven't changed content-wise.

This may not be that obvious if you're only building your image once, but we're running acceptance tests which create containers repeatedly, and we *do notice* if it does unecessary work every time.

This led me to concoct the following, beautiful hack:

```yaml
checkout:
  post:
    - find . -exec touch -t 201401010000 {} \;
    - for x in $(git ls-tree --full-tree --name-only -r HEAD);\
      do touch -t \
      $(date -d "$(git log -1 --format=%ci "${x}")" +%y%m%d%H%M.%S) \
      "${x}"; done
```

This goes through the repo after checkout, first resetting every the timestamp for each item to 2014/01/01. Then, for all files which are tracked by git, we take the timestamp of the last commit that changed this file.
Not beautiful, and there might be a better solution, but CircleCI's support at the time of writing couldn't come up with anything better on the spot.

## Testing
The actual tests are easy. We've got full-fledged containers disposable, and we add the following sections to `circle.yml`:

```
test:
  override:
    - docker run "cockroachdb/cockroach-dev" test
    - make acceptance
```

The former runs our unit tests and does race testing, while the latter boots up a cluster of three nodes and verifies that it connects fully. We'll make sure to add more tests here over time and verify complex behaviour of the cluster as a system.

## Deployment

Finally, we can automagically deploy the new build in case the tests come up green. `./build/build-docker-deploy` builds our deployment container; the rest is straight from CircleCI's examples.

With this section, any green build will be build statically, tested again for good measure in the static environment, and finally pushed to the Docker registry for you all to enjoy.

```
deployment:
  docker:
    branch: master
    commands:
      - ./build/build-docker-deploy.sh:
         timeout: 600
      - sed "s/<EMAIL>/$DOCKER_EMAIL/;s/<AUTH>/$DOCKER_AUTH/" > \
        "resources/deploy_templates/.dockercfg.template" > ~/.dockercfg
      - docker push cockroachdb/cockroach
```

## You can't delete images
Just something to think about. It's tempting to create containers that have predefined names, and sometimes you'll want to delete and recreate them in your tests. Doesn't work here.
[CircleCI](http://circleci.com)'s support told me that

> deleting btrfs snapshots requires additional capabilities that the
> pseudo-root user inside a container doesn't have. Generally docker rm isn't
> necessary. We completely destroy the entire container and everything associated with it at the end of a build anyway.


You can work around this usually, but if you write more complex acceptance tests, just make sure that you never need to remove a container in the process.

## The full circle.yml & conclusion

* [CircleCI](http://circleci.com) is great and the folks over there are
  helpful. Also: Free for Open Source projects.
* [Docker](http://docker.io) support works well and opens the door to
  testing a lot more than you could before.
* Caching is a bit tricky: Large images cost time to move around the network,
  and they need to be fed in and out of Docker manually - also cost-intensive.
  Apparently, solutions for this are being investigated.

All in all: It's already good, and it will be much better soon. There's really no reason to go without Docker if you already have a Dockerfile for your project.

Here's our full Dockerfile at the time of writing (slightly edited):

```yaml
machine:
  services:
    - docker

checkout:
  post:
    - find . -exec touch -t 201401010000 {} \;
    - for x in $(git ls-tree --full-tree --name-only -r HEAD); do \
      touch -t $(date -d "$(git log -1 --format=%ci "${x}")" \
      +%y%m%d%H%M.%S) "${x}"; done

dependencies:
  cache_directories:
    - "~/docker"
  override:
    - mkdir -p ~/docker
    - if [[ -e ~/docker/base.tar ]]; then \
        docker load -i ~/docker/base.tar; fi
    - ./build/build-docker-dev.sh
    - docker save "cockroachdb/cockroach-devbase" > ~/docker/base.tar
    - if [[ ! -e ~/docker/dnsmasq.tar ]]; then \
        docker pull "cockroachdb/dnsmasq" && \
        docker save "cockroachdb/dnsmasq" > ~/docker/dnsmasq.tar; \
      else docker load -i ~/docker/dnsmasq.tar; fi
    - docker history "cockroachdb/cockroach-dev"

test:
  override:
    - docker run "cockroachdb/cockroach-dev" test
    - make acceptance

deployment:
  docker:
    branch: master
    commands:
      - ./build/build-docker-deploy.sh:
          timeout: 600
      - sed "s/<EMAIL>/$DOCKER_EMAIL/;s/<AUTH>/$DOCKER_AUTH/" < \
        "resources/deploy_templates/.dockercfg.template" > ~/.dockercfg
      - docker push cockroachdb/cockroach
```

You can find the setup as described above in [Cockroach@7e452](https://github.com/cockroachdb/cockroach/tree/7e452c2a8a9537e4ba5906ccc8a54c50b06885fb).
