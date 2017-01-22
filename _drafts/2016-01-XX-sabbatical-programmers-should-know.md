---
layout: post
title: "Sabbatical Reading List: What Every Programmer Should Know"
comments: true
permalink: "programmers-should-know"
---

### What Every Programmer Should Know About Memory (and Futexes)

Let's face it, the following was rarely ever muttered:

    "I wonder whether this object fits in a cache line" -- A Ruby programmer

This [well-known classic](https://sploitfun.wordpress.com) by Ulrich Drepper
gives the lay of the land of how modern computers store and, most importantly,
cache your precious variables and data structures and how the many CPUs we have
in our computers these days deal with these caches.

Since the various methods of retrieving data vary in latency usually by orders
of magnitude, code which needs to run fast needs to optimize for the reality
underlying its data.

It's interesting to note that the imaginary Ruby programmer quoted above is
unlikely to *actually* need to know about memory, seeing how slow the
interpreted code already is and how little it matters in many scenarios.
Clearly, Ulrich Drepper pictures a programmer as a more badass, low level
performance-hunting beast than most of "us" actually are.

Still, a true programmer should fight their imposter syndrome and know (or
learn) the things conveyed therein (and might also want to check
out [Futexes are tricky](#TODO) because it's fun to read).

And it never hurts to know that the basic `O(1)` assumption for basic
operations doesn't hold in general and even the runtime of quicksort [can
be dominated by something as low-level as branch mispredictions](http://algo2.iti.kit.edu/sanders/papers/KalSan06.pdf).

The crowd really thins out as you go low. To give another example, few people
fully grok modern schedulers completely; from the famous [You are not expected
to understand
this](https://en.wikipedia.org/wiki/Lions'_Commentary_on_UNIX_6th_Edition,_with_Source_Code#.22You_are_not_expected_to_understand_this.22)
in 1976 to the recent discovery that for years and years, the Linux scheduler
has performed sub-par, resulting in [a Decade of Wasted
Cores](https://www.ece.ubc.ca/~sasha/papers/eurosys16-final29.pdf).
