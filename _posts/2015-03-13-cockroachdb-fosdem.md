---
layout: post
title: "Cockroach at FOSDEM 2015"
comments: true
permalink: "cockroachdb-talk-fosdem-2015"
---

At this year's edition of [FOSDEM](http://fosdem.org), I gave a presentation
on [Cockroach](http://github.com/cockroachdb/cockroach) in the [Go Devroom](https://fosdem.org/2015/schedule/track/go/).

[Cockroach](http://github.com/cockroachdb/cockroach) aims to provide a
transactional, scalable, and available datastore for everyone - basically
a database that feels as solid as a traditional SQL server, but scales out and
replicates data effortlessly (which, of course, is not true with traditional
RDBMS) while avoiding the pitfalls that traditionally come with
[NoSQL](http://en.wikipedia.org/wiki/NoSQL) datastores.

In the talk, I put a focus on providing technical details about our
implementation of fast, lock-free transactions to give the audience an
intuitive understanding of what's happening behind the scenes.

You can find out for yourself how well that worked by giving the video
below a try, and by checking out
[Cockroach](https://github.com/cockroachdb/cockroach) if you're curious to
learn more.

<iframe width="560" height="315" src="https://www.youtube.com/embed/ndKj77VW2eM?list=PLtLJO5JKE5YDK74RZm67xfwaDgeCj7oq" frameborder="0" allowfullscreen></iframe>

Thanks to everyone who worked towards making [FOSDEM](http://fosdem.org) and
the [Go Devroom](https://fosdem.org/2015/schedule/track/go/) such a pleasant
experience.
