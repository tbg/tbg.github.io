---
layout: post
title: "Cockroach at the NoSQL User Group Cologne"
comments: true
permalink: "cockroachdb-talk-nosql-user-group-cologne"
---

I've spent quite some time this year working on
[Cockroach](https://github.com/cockroachdb/cockroach),
a [NewSQL](http://en.wikipedia.org/wiki/NewSQL) database similar in spirit to
[Google's Spanner](http://en.wikipedia.org/wiki/Spanner_%28database%29). It is
a distributed, replicated and available database that comes with transactions
as one of its key features. The project is extremely ambitious but the basic
design is sleek and beautiful.

Recently, I talked about Cockroach in the [NoSQL User Group
Cologne](http://www.meetup.com/NoSQL-Usergroup-Cologne/) and explained, among
other things, that you actually want and are able to have transactions in your
distributed database in many situations, and that you can have something like
[Spanner](http://en.wikipedia.org/wiki/Spanner_%28database%29) without the infrastructure and hardware zoo.

If you're curious, check out
[Cockroach](https://github.com/cockroachdb/cockroach) and watch the talk
below (starts at about 1m15s in):

<iframe width="560" height="315" src="https://www.youtube.com/embed/jI3LiKhqN0E?t=1m20s" frameborder="0" allowfullscreen> </iframe>
