---
layout: post
title: "Golang: Statically linked binary and tests for Cockroach"
comments: true
permalink: "linking-golang-go-statically-cgo-testing"
---

*At [Cockroach](https://github.com/cockroachdb/cockroach), we write a lot of tests, which is great and absolutely necessary: Building a CP ([consistent and partition-tolerant](http://en.wikipedia.org/wiki/CAP_theorem)) distributed system (like [Cockroach](https://github.com/cockroachdb/cockroach)) without getting everything exactly right just gives you another system without any guarantees. Below I'll describe what I did to build both our main binary and our tests (!) statically, and how far I got.*

  **Edit (04/11/2015)**: Ever since Go 1.4 came around, this article has been slightly outdated. A change in 1.4 altered the behaviour of the `-a` flag such that it would not rebuild the standard library. Consequently, the `netgo` tag did not have the desired effect any more. There are various discussions about this to be found online, and luckily there's an easy fix: just add the `-installsuffix netgo` parameter to your go build flags. That causes the packages to be built in `${GOROOT}/pkg/<arch>_netgo` instead, causing the `-a` flag to behave as it should.


**TL;DR: `-a` is surprisingly useful.**

Our codebase is mostly [Go](http://golang.org/), but with a number of references to C++. The result of a successful build is a single binary, and we want users to be able to deploy this binary right away without having to setup a build toolchain or making sure that they have the right libs we link against.

In a nutshell, we want a statically linked binary. And **even more**, we want to compile the **tests** as statically linked binaries so that they can be run in their final deploy environment. It is this last part that makes things a bit more complex.

## A lot of Go links dynamically
Often, you hear complaints about Go binary sizes along with the explanation that *it's statically linked, always*. We'd be happy to have it that way, but usually only *simple* Go programs are statically linked - they can use the gc tool chain (5l, 6l, 8l), and even just using the `net` package requires a special build tag (-tags netgo), or you end up with a dynamically linked binary.

And once you're reaching out to the C/C++ world via CGO, `go build` will definitely use the external linker, and you're very likely ending up with a **dynamically linked** binary.

At [Cockroach](https://github.com/cockroachdb/cockroach), we certainly have such dependencies: Our own C++ libs used for low-level stuff, protocol buffers and [RocksDB](http://rocksdb.org/), a more performant fork of [Google's LevelDB](http://en.wikipedia.org/wiki/LevelDB). Those dependencies offer static libraries, of course, so when I started this I was reasonably optimistic.

### What's being linked?
Let's start with the build process that we use when developing - we don't care about the way it's linked, and consequently we (more or less) run

```
$ go build -i -o cockroach
$ ldd cockroach
linux-vdso.so.1 =>  (0x00007fffd6ffe000)
libstdc++.so.6 => /usr/lib/x86_64-linux-gnu/libstdc++.so.6 (0x00002ab74bab6000)
libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00002ab74bdba000)
librt.so.1 => /lib/x86_64-linux-gnu/librt.so.1 (0x00002ab74c0c0000)
libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00002ab74c2c8000)
libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00002ab74c4e6000)
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00002ab74c6fc000)
/lib64/ld-linux-x86-64.so.2 (0x00002ab74b891000)
```
Yep, as expected, a lot of external libraries are being referenced.

### Statically, first try
Some light googling brings up [this article](http://blog.hashbangbash.com/2014/04/linking-golang-statically/), so we try this next:

```
$ go build --ldflags '-extldflags "-static"'
...
In function `std::tr1::__detail::_Prime_rehash_policy::_M_need_rehash(unsigned long, unsigned long, unsigned long) const':
/usr/include/c++/4.8/tr1/hashtable_policy.h:469: undefined reference to `ceilf'
/usr/include/c++/4.8/tr1/hashtable_policy.h:475: undefined reference to `ceilf'
/home/tobias/Code/Go/src/github.com/cockroachdb/cockroach/_vendor/usr/lib/libprotobuf.a(descriptor.o):
In function `lower_bound<long unsigned int const*, float>':
...
```

*Ouch*. Well, thinking about it, we probably have to tell the linker some of the libs it should link against.

### Adding libraries
 `ceilf` sounds mathy, so let's add the `math` library (`-lm`):

``` 
$ go build --ldflags '-extldflags "-lm -static"'
...
/usr/include/c++/4.8/tr1/hashtable_policy.h:384: undefined reference to `std::tr1::__detail::__prime_list'
/home/tobias/Code/Go/src/github.com/cockroachdb/cockroach/_vendor/usr/lib/libprotobuf.a(dynamic_message.o): In function `lower_bound<long unsigned int const*, float>':
/usr/include/c++/4.8/bits/stl_algobase.h:965: undefined reference to `std::tr1::__detail::__prime_list'collect2: error: ld returned 1 exit status
/usr/local/go/pkg/tool/linux_amd64/6l: running gcc failed: unsuccessful exit status 0x100
make: *** [build] Error 2
```
Well. The math errors are gone, but there's still some other stuff. Looks like it wants to be linked against stdlib? We can add that statically via `-lstdc++`. Another try:

```
$ go build --ldflags '-extldflags "-lm -lstdc++ -static"
# github.com/cockroachdb/cockroach
...00003.o: In function `_cgo_0db69e1b5445_Cfunc_getpwnam_r':
(.text+0x58): warning: Using 'getpwnam_r' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
...00003.o: In function `_cgo_0db69e1b5445_Cfunc_mygetpwuid_r':
.../go/src/pkg/os/user/lookup_unix.go:27: warning: Using 'getpwuid_r' in statically
linked applications requires at runtime the shared libraries from the glibc version used for linking
...000001.o: In function `_cgo_14616c423265_Cfunc_freeaddrinfo':
.../go/src/pkg/net/cgo_unix.go:97: warning: Using 'getaddrinfo' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
$ echo $?
0
ldd cockroach
     not a dynamic executable
```

### Success! ...oh wait.

*That wasn't so hard. Some warnings remain, but we have an executable.*

Fast forward a little: It turns out that the binary created in this way has no functional networking capabilities. Looking at the last warning about `getaddrinfo`, this seems to make sense - `getaddrinfo` supposedly tries to get to glibc at runtime, which isn't available and will hence return an error. If it can't connect anywhere, it's probably not such a great fit for a **distributed** database.

Luckily, Go has recently (in 1.2) acquired a new build tag `netgo` which addresses precisely this issue:

```
The net package requires cgo by default because the host operating system must in general mediate network call setup. On some systems, though, it is possible to use the network without cgo, and useful to do so, for instance to avoid dynamic linking. The new build tag netgo (off by default) allows the construction of a net package in pure Go on those systems where it is possible.
```

So this is exactly up our alley, and since we're just fooling around at this point, let's not think about whether going down this road will impact networking performance negatively. We've come too far to stop.

```
$ go build -tags netgo --ldflags '-extldflags "-lm -lstdc++ -static"' -i -o cockroach
...
exact same stuff as before
```

Yep, exact same stuff. And exact same problems. In particular, the warning about `getaddrinfo` is still there, and that's an indicator that networking still leaves Go and will fail for us.

### The magic flag
The next step I found out about the hard way: By trying out a lot of random things.
My solution: add the `-a` flag to `go build`:

```
-a  force rebuilding of packages that are already up-to-date.
```

**Yes. Adding this tag suddenly causes the `-tags netgo` parameters to do their job.** That's all I can tell you.

```
go build -a -tags netgo --ldflags '-extldflags "-lm -lstdc++ -static"' -i -o cockroach
# github.com/cockroachdb/cockroach
/var/tmp/go-link-8PWBSa/000002.o: In function `mygetpwuid_r':
/usr/local/go/src/pkg/os/user/lookup_unix.go:72: warning: Using 'getpwnam_r' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
/var/tmp/go-link-8PWBSa/000002.o: In function `_cgo_0db69e1b5445_Cfunc_mygetpwuid_r':
/usr/local/go/src/pkg/os/user/lookup_unix.go:27: warning: Using 'getpwuid_r' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
```
The warnings left above tell us that we won't be able to use the `os/user` package to look up system user accounts. I've checked manually that those calls simply *fail* and don't *segfault*, so *whatever*. Of course they pop up with every build, so ideally we want to get rid of this eventually, but right now let's continue down the road.

### Intermediate result
Ok, we've got a binary that seems to be working. I can put it into a [Docker container](http://docker.io/) based on [BusyBox](http://www.busybox.net/), start a couple of nodes and they can connect to each other - proving that networking is *at least kind of* working.

But, we've seen above that getting a binary doesn't mean that binary is worth anything - so let's get to doing the same for tests!

### Statically linked tests - surprise!
Let's use what we've learnt about the build process and hope for the best. We are building the tests for Cockroach's `gossip` package, and run

```
$ go test -a -tags netgo -ldflags '-extldflags "-lm -lstdc++ -static"' -c "./gossip" -logtostderr -timeout 10s
$ ldd gossip.test
linux-vdso.so.1 =>  (0x00007fff4d5fe000)
libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00002aca89834000)
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00002aca89a52000)
/lib64/ld-linux-x86-64.so.2 (0x00002aca8960f000)
```

**Oh, come on**. So apparently you can tell the linker what you want, you might still end up with a *dynamically linked binary*, at least when compiling tests. Presumably, Go internally enriches your stuff with its test runner, and something you don't want happens.

You *could* add the `-x` flag to see exactly what the Go toolchain does, but, well, I've tried that and I couldn't whole-heartedly tell you that it has improved my understanding of things.

The second instinct: Slap some libs on top of it - adding a `-lpthread` to get `libpthread` could be fun. But, libpthread is part of GLIBC, so apparently that's something you oughtn't be doing anyways. Luckily, you don't have to, it won't change a single thing. There's one thing that does, however:

## Disabling CGO where not required
I've mentioned previously that in this project, we're using CGO to reach out to C++-Land. Interestingly, there's something there that saves the day for us: A couple of packages end up compiling their tests statically if we do it as above, and a couple won't.

Fortunately, those packages that actually **need** CGO end up statically linked, and only packages that **don't require it** have the issue above. So, time for a hack, straight from our [Makefile](https://github.com/cockroachdb/cockroach/blob/7e452c2a8a9537e4ba5906ccc8a54c50b06885fb/Makefile):

```bash
for p in github.com/cockroachdb/cockroach/gossip; do \
  NAME=$(basename "$p"); \
  OUT="$NAME.test"; \
  DIR=$(go list -f \{\{.Dir\}\} ./...$NAME); \
  go test  -a -tags netgo -ldflags '-extldflags "-lm -lstdc++ -static"' -c "$p" -logtostderr -timeout 10s || break; \
  if [ -f "$OUT" ]; then \
    if [ ! -z "1" ] && ldd "$OUT" > /dev/null; then \
    2>&1 echo "$NAME: rebuilding with CGO_ENABLED=0 to get static binary..."; \
    CGO_ENABLED=0 go test  -a -tags netgo -ldflags '-extldflags "-lm -lstdc++ -static"' -c "$p" -logtostderr -timeout 10s || break; \
    fi; \
    mv "$OUT" "$DIR" || break; \
  fi \
done
```

At this point, I couldn't say I'm exactly happy with it, but **at least I got what I wanted**. Statically linked test binaries that I can slap anywhere the cockroach binary would run.

## Race tests?

Nope. `runtime/race` won't build for those tests that require `CGO_ENABLED=0` above, such as our `gossip` package. You end up with

```
gossip: rebuilding with CGO_ENABLED=0 to get static binary...
# testmain
runtime/race(.text): undefined: __libc_malloc
runtime/race(.text): undefined: getuid
runtime/race(.text): undefined: pthread_self
runtime/race(.text): undefined: madvise
```

so I think this is where we can't go further at this point.

### Summary
I basically got what I wanted but at the expense of a few hacks:

* use `-a` flag for its desired (but mystery) side-effect
* ignore warnings from package `os/user` - user lookups will always fail
* have to use `-tags netgo` - performance?
* get lucky with CGO_ENABLED=0

It seems to me that building static tests isn't something that was given a lot of thought; especially for race tests this seems fairly hard to get.

And that concludes this survey of building things statically for a non-trival Go project.
The state of things described in this post is contained in [Makefile@7e452](https://github.com/cockroachdb/cockroach/blob/7e452c2a8a9537e4ba5906ccc8a54c50b06885fb/Makefile) and will likely see a lot of changes over time.
