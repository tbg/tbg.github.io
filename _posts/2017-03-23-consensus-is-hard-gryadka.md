---
layout: post
title: "Gryadka is not Paxos, so it's probably wrong (RETRACTED)"
comments: true
permalink: "if-its-not-paxos-its-probably-wrong-gryadka"
---

*[Leslie Lamport][lamport] purportedly once said (though I can't prove it):*

> If it's not Paxos, it's probably wrong.

In the original version of this post, I claimed that [gryadka][gryadka], which
claims to be built on top of a Paxos-backed CAS register, was incorrect by
providing a "counter-example".

It turns out that you really drop 20 IQ points on a vacation, and that I had
not at all provided a counter-example, despite having looked closely at the
algorithm.

My claim that the Paxos-style CAS register exhibited anomalies is thus wrong
(or, at least, unproven).

My apologies to [@rystsov][rystsov-tw], who kindly pointed out my mistake. I'll
look into the algorithm more (post-vacation), but now considering that it might
be correct, which would be quite exiting - perhaps there are Paxos-like things
which are not Paxos but are not wrong.

The original version of this post is available in the [commit history][ch],
preserved for posterity. My mistake not carrying out the read phase in the
middle fully: If you do, then you'll commit the old value "again", making the
anomaly-exhibiting later CAS fail.

[lamport]: http://lamport.azurewebsites.net
[rystsov-tw]: https://twitter.com/rystsov
[gryadka]: https://github.com/gryadka/js
[ch]: https://github.com/tschottdorf/tschottdorf.github.io
