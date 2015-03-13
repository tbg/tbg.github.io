---
layout: post
title: "Docker at Cockroach: the power of development and deployment containers"
comments: true
permalink: "docker-cockroach-development-deployment-containers"
---

*At [Cockroach](https://github.com/cockroachdb/cockroach), we are building a scalable, distributed database and use [Docker](http://docker.io) to automate and streamline our builds, tests and deploy processes.
When you read about Docker, you normally run across simple Dockerfiles. In this post, I'll introduce some of the more advanced constructs we are using (nothing fancy - but useful), and how powerful they are in making our lives easier. In particular, you'll learn how developers can work on [Cockroach](https://github.com/cockroachdb/cockroach) without installing anything, and how we reliably build the same deployment-ready minimal containers on any system.*

Concisely put, Cockroach (at the time of writing) comes with three images:

* `cockroachdb/cockroach-devbase`: A development base image.
  It does not contain any of the actual Cockroach code, but the build toolchain and vendored-in versions of all the dependencies. Normally, if you work on the code, you don't need to change this image. We factored it out because this image is a good point to base your build on and can be cached nicely in continuous integration (we use the awesome [CircleCI](http://circleci.com)).
* `cockroachdb/cockroach-dev`: The development image.
  This is a simple image based on the cockroach-devbase, with only a few lines of Dockerfile. 
* `cockroachdb/cockroach`: The deployment image, i.e. the one you use if you're
  not actually developing, but rather running a Cockroach cluster.
  This one is based on [BusyBox](http://www.busybox.net/) and comes with a more complicated build process:
A development container builds statically linked binaries and tests. The main binary is put into a minimal container, and the tests can be mounted into that container to check the functionality.

## The Development Image `cockroachdb-dev`

In short, the [Cockroach code base](https://github.com/cockroachdb/cockroach) is [Go](http://golang.org/) plus some C++. Without Docker, if you were to clone our repo and tried to hack on it, you'd have to do the following:

```bash
$ git clone git@github.com:/cockroachdb/cockroach
# ...
$ cd cockroach; ./bootstrap.sh
# ...
# pages and pages of libraries being built
# ...
# 10 minutes later:
$ make build
# ...
$ ./cockroach
Usage:

        cockroach command [arguments]
# ...
```

and that is the best-case scenario in which you have the appropriate new version of C++ installed to build [RocksDB](http://rocksdb.org) and the various other dependencies, such as `snappy`, `protobuf`, `bz2`, ... didn't end up having any build hiccups.

Also, we would have to script variants of these processes for our continuous integration based on the test environment that they provide.

That can all be done and in fact I personally hack on Cockroach outside of Docker, but as an outsider wanting to contribute, I don't want them to deal with all of that.

Instead, they could simply do a quick `docker build -t "cockroachdb/cockroach-dev" .` from our main repo. Let's try this with a fresh clone:

```bash
$ docker build -t "cockroachdb/cockroach-dev"
Sending build context to Docker daemon  58.1 MB
Sending build context to Docker daemon
Step 0 : FROM cockroachdb/cockroach-devbase:latest
Pulling repository cockroachdb/cockroach-devbase
0407ad0fa9fd: Pulling image (latest) from cockroachdb/cockroach-devbase, endpoint: https://registry-1.d0407ad0fa9fd: Download complete
511136ea3c5a: Download complete
# [...]
8d37c4dafcdf: Download complete
Status: Downloaded newer image for cockroachdb/cockroach-devbase:latest
 ---> 0407ad0fa9fd
Step 1 : MAINTAINER Tobias Schottdorf <tobias.schottdorf@gmail.com>
 ---> Running in e6528c82806a
 ---> 768b9e8f0212
Removing intermediate container e6528c82806a
Step 2 : ADD . /cockroach/
 ---> d94b378cfa97
Removing intermediate container 2a3b51d06418
# ...
go build  -i -o cockroach
 ---> 2fd9e266b824
Removing intermediate container 0b156cca7c81
Step 6 : EXPOSE 8080
 ---> Running in 4c99f7e9fa84
 ---> 85d3a50c9f07
Removing intermediate container 4c99f7e9fa84
Step 7 : ENTRYPOINT /cockroach/cockroach.sh
 ---> Running in 6ccf7d7498df
 ---> 7f811d0fac6f
Removing intermediate container 6ccf7d7498df
Step 8 : CMD --help
 ---> Running in 24de6f231385
 ---> 0065234019a6
Removing intermediate container 24de6f231385
Successfully built 0065234019a6
```

After the download of the `cockroachdb/cockroach-devbase` image, it took about 20 seconds to build the `cockroachdb/cockroach-dev` image. Now, right away, we can run cockroach:

```bash
$ docker run "cockroachdb/cockroach-dev" --help
Usage:

        cockroach command [arguments]
# ...
```

After perhaps making some local changes to the git repo, another `docker build -t "cockroachdb/cockroach-dev"` will update the image, taking virtually as long as it would outside of the virtual environment, and running the tests is as easy as

```bash
$ docker run "cockroachdb/cockroach-dev" test
# ...
go test  -run ".*" "./..." -logtostderr -timeout 10s
ok      github.com/cockroachdb/cockroach/client 0.216s
# ...
go test  -race -run ".*" "./..." -logtostderr -timeout 1m
ok      github.com/cockroachdb/cockroach/client 1.811s
# ...
```

We can also run `docker run -t -i "cockroachdb/cockroach-dev" shell`, which is a special parameter that drops us into a shell. Since the environment variables have all been changed to point to the vendored libraries, this is a good setting to invoke `go` directly without going through `make` - something that may be useful to do for different reasons.

That's good stuff, and a fairly standard use of Docker at this point.

## The Base Image `cockroach-devbase`

The base image is what the previous image was based on - it's fairly simple in itself, too. Omitting some less interesting bits, it boils down to the following Dockerfile:

```docker
FROM golang:latest

MAINTAINER Tobias Schottdorf <tobias.schottdorf@gmail.com>

# Setup the toolchain.
RUN apt-get update -y && apt-get dist-upgrade -y && \
 apt-get install --no-install-recommends --auto-remove -y git build-essential pkg-config file &&\
 apt-get clean autoclean && apt-get autoremove -y && rm -rf /tmp/* /var/lib/{apt,dpkg,cache,log}

ENV GOPATH /go

# ...

ADD . /cockroach/build/devbase/
RUN ["./build/devbase/godeps.sh"]
RUN ["./build/devbase/vendor.sh"]

CMD ["/bin/bash"]
```

Basing itself of `golang:latest`, we end up not having to worry about Go, and then it's a simple matter of having a working compiler environment (golang:stable is Jessie, so it ships with a recent GCC), pulling the go dependencies and vendoring the required libraries into the tree.

## The Deployment Image `cockroach`

This is where things get fun. If you want to build a deploy image, you need a little shell scripting on top of calling Docker.

The deploy image creation script requires a working cockroach/cockroach-dev image, in which a statically linked (linux64) binary is built. Using this binary, a deploy image based on BusyBox is created.

Additionally, we built statically linked tests which are mounted into the appropriate location on the deploy image, running them once. These are not a part of the resulting image but make sure that at least on the machine that creates the deploy image, the tests all pass.

So you get both upsides: A small image, containing only BusyBox and Cockroach's main binary - but you can run all the tests as well if you supply them to the container from the outside file system. The resulting script looks something like this:

```bash
#!/bin/bash
# Build a statically linked Cockroach binary
# Author: Tobias Schottdorf (tobias.schottdorf@gmail.com)
set -ex
cd -P "$(dirname $0)"
DIR=$(pwd -P)
rm -rf resources cockroach .out && mkdir -p .out
docker run "cockroachdb/cockroach-dev" shell "export STATIC=1 && \
  cd /cockroach && (rm -f cockroach && make clean build testbuild) >/dev/null 2>&1 && \
  tar -cf - cockroach \$(find . -name '*.test' -type f -printf '"%p" ') resources" \
> .out/files.tar;
```

The above builds the static binary and tests inside of the container, collects them
from the file system, tars them and sends them out to stdout.

Next the script extracts everything, puts the `cockroach' binary in the right spot and builds the BusyBox container:

```bash
tar -xvC .out/ -f .out/files.tar && rm -f .out/files.tar
mv .out/cockroach .
cp -r .out/resources ./resources
docker build -t cockroachdb/cockroach .
```

Test run: Mounting the .out directory into the container's /test/, the
container will run all the tests, making sure that everything looks good.

```bash
docker run -v "${DIR}/.out":/test/.out cockroachdb/cockroach
```

The actual Dockerfile that is being built is trivial:

```docker
FROM busybox:buildroot-2014.02

MAINTAINER Tobias Schottdorf <tobias.schottdorf@gmail.com>

RUN mkdir -p /test /cockroach
ADD cockroach /cockroach/
ADD cockroach.sh /cockroach/
ADD test.sh /test/

EXPOSE 8080
ENTRYPOINT ["/cockroach/cockroach.sh"]
```

With this build process, the actual deployment container is relatively small. The `cockroach` binary weighs in at about 44mb at the time of writing; BusyBox adds another meager 5mb.

Compared to the >1GB `cockroachdb/cockroach-dev` image, this is very convenient.

You can find the whole shebang in [Cockroach@7e452](https://github.com/cockroachdb/cockroach/tree/7e452c2a8a9537e4ba5906ccc8a54c50b06885fb) - it's quite likely that over time, things will change quite a bit still.
