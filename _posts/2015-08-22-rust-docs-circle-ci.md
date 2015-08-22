---
layout: post
title: "Rust and automatically generating docs on CircleCI"
comments: true
permalink: "rust-docs-circle-ci"
---

After some (private) dabbling around with [Rust](http://rust-lang.org), I now
have the first public toy project: [hlc-rs](https://crates.io/crates/hlc), a
[hybrid logical clock](http://muratbuffalo.blogspot.com/2014/07/hybrid-logical-clocks.html).

I'm going to omit the customary ode of joy to the Rust language, tooling, and [the community](https://github.com/nrc/rustfmt/issues/197) and only share a small snippet
that I hacked together to automatically publish the [auto-generated documentation](https://doc.rust-lang.org/book/documentation.html) via [github pages](https://pages.github.com)
after a successful `master` test run on [CircleCI](https://circleci.com).

# Step 1

I'm assuming the repository root is your crate's root directory. I'm using this
`circle.yml`:

```yaml
dependencies:
  post:
    - git config --global user.email my@email.com
    - git config --global user.name "My Name"
    - curl -sf -L https://static.rust-lang.org/rustup.sh | sh /dev/stdin --channel=nightly --yes
test:
  override:
    - cargo test

deployment:
  docs:
    branch: master
    commands:
      - cargo doc && git fetch origin gh-pages && git checkout gh-pages && (git mv doc doc-$(git describe --always master^) || rm -rf doc) && mv target/doc/ ./doc && git add -A ./doc* && git commit -m 'update docs' && git push origin gh-pages
```

Pretty standard (also, likely suboptimal - but that's fine for a project that
doesn't see a lot of traffic). The notable part is the last one:

```bash
# Generate the documentation.
cargo doc \
  && git fetch origin gh-pages \ # make sure we have the branch
  && git checkout gh-pages \     # check it out
  # If there's a `doc` directory, move it away or delete it if that fails.
  && (git mv doc doc-$(git describe --always master^) || rm -rf doc) \
  # Move the new docs in their place.
  && mv target/doc/ ./doc \
  # Add both the old docs and the new one.
  && git add -A ./doc* \
  # Commit, duh.
  && git commit -m 'update docs' \
  # Push new commit.
  && git push origin gh-pages
```

For setting this up, three simple steps are needed:

* add the project to [CircleCI](http://circleci.com),
* [add read-write deploy key for GitHub and CircleCI](https://circleci.com/docs/adding-read-write-deployment-key), and
* push an initial `gh-pages` branch:

  ```bash
  git checkout --orphan
  git reset
  git commit --allow-empty -m 'initial commit'
  vi circle.yml # see below
  git push origin gh-pages
  ```

  where `circle.yml` is a dummy (so that the branch doesn't give you test failures):

  ```yaml
  test:
    override:
      - echo "noop"
  ```

Of course, the same easily works on other CI platforms and possibly there are
[other ways to do it](https://www.reddit.com/r/rust/comments/3e1xgy/how_do_you_folks_autogenerate_the_doc_pages_for/).

That's it already! Now [documentation like this](https://tschottdorf.github.io/hlc-rs/doc/hlc/) should be auto-generated for you with the next CI run on `master`.
