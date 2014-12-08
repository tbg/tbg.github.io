---
layout: post
title: "Quickly cd'ing to local git repositories"
comments: true
permalink: "cd-git-repository-helper"
---

*When I work on my laptop, almost exclusively that work happens inside of a [Git repository](http://en.wikipedia.org/wiki/Git_%28software%29). Working with [Go](http://golang.org), that usually means typing something along the lines `cd ~/Code/Go/src/github.com/cockroachdb/cockroach`. If you're changing repos a lot, that can be annoying.*

But it's easy to optimize this a little bit, because *usually you only have one copy of the repository on your file system*.

With the help of the script below,

```bash
$ cd ~/Code/Go/src/github.com/cockroachdb/cockroach
...
$ cd ~/dotfiles
...
$ cd ~/Code/tschottdorf.github.io
...
$ cd ~/Code/Go/src/github.com/jpetazzo/docker-busybox
```

turns into

```bash
$ gitcd cockroach
...
$ gitcd dotfiles
...
$ gitcd tscho*
...
$ gitcd *busybox
```

which is as efficient as it gets. I've topped this with

* `alias gcd="gitcd"`
* `alias ccd="gitcd cockroachdb/cockroach"`

to optimize even further. Here's the script:

```bash
#!/bin/bash

function gitcd () {
  if [ -z "$1" ]; then
    return 1
  fi
  NEWDIR=$(find ~ -path "*/$1/.git" -a -type d -exec echo "{}/.." \; | head -n 1)
  if [ -z "${NEWDIR}" ]; then
    return 1
  fi
  cd -P "${NEWDIR}"
  echo "$(pwd -P)"
}
```

It doesn't do much except for invoking `find`, looking for the first occurance of a folder that matches your input and contains a `.git` subfolder. If there is one, simply `cd` there. I'm sure somebody out there is already using something like this - it's just so trivial but useful.

The original script (which may see some future tweaks as I find myself wanting them) is in my [git-cd repository](https://github.com/tschottdorf/git-cd).
