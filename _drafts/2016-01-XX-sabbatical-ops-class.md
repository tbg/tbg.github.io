---
layout: post
title: "Sabbatical Reading List: Geoffrey Challen's ops-class"
comments: true
permalink: "sabbatical-ops-class"
---

The [ops-class](https://www.ops-class.org/) online lecture series is a free
introduction to operating systems, but in particular kernel hacking. Developed
and taught by [Geoffrey Challen](https://blue.cse.buffalo.edu/people/gwa/) at
University of Buffalo (though it appears he [will be moving
elsewhere](https://www.bluegroup.systems/posts/2016-10-22-the-best-way-to-not-get-tenure/)),
this material stands out because of how hands-on and challenging it is.

As you'd expect, you get all of the lectures taped along with the slides (which
is what I consumed, as they're more readily consumed at my own pace plus work
better offline), but it doesn't stop there.

Building on MIT's teaching OS [OS/161](http://os161.eecs.harvard.edu/) and
associated simulator, the assignments have you hack on the rudimentary OS/161
kernel and add vital components, automatically test your results, and even let
you *benchmark* your solutions against others'.

For example, one of the first real exercises is implementing basic
synchronization primitives, followed by adding support for user processes and,
finally, virtual memory. All of these are far from trivial and increasing in
complexity, so solving them is a real achievement and probably the closest you
can get to "real" kernel programming, just shy of writing your own kernel
(which can be observed in, for example, [Philipp Oppermann's blog on writing
a kernel in Rust](http://os.phil-opp.com/)) or contributing to the actual Linux
kernel.

The slight downside to `ops-class` is that at the time of writing, pull
requests to the material have been [sitting around for
months](https://github.com/ops-class/vagrant/pull/9), so make sure to grab
these unmerged fixes to avoid [wasting
time](https://github.com/ops-class/vagrant/issues/13).
