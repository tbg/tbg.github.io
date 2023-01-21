---
title: "I knew about the sharp nail and it got me anyway"
date: 2023-01-13T08:02:40+01:00
---

I spent the entirety of last morning dealing with the fallout of incorrectly
handling a piece of largely unnecessary complexity in the CockroachDB code base.
I was familiar with the complexity; I was around when the complexity was
introduced many years ago; I field questions (by others working on CockroachDB)
about it a few times a year; I was, yesterday morning, working specifically in
an area touched by the complexity and in fact had *just* improved its code
documentation. And yet, it got me and I wasted a few hours, and not for the
first time either. So it seems guaranteed to happen again, and again, and again.

It's not easy to fix this bit of complexity. It would be somewhat invasive.
So you'd need some sort of indication that it would be worth it.

I tried to quantify the cost of this particular misshapen piece of (largely
accidental) design. Maybe I could see it come up repeatedly in the commit log,
in Slack, etc, and eyeball how many engineering hours were spent each time?

One

```
git log --pickaxe-regex -S '\\x02|LocalMax'
```

`git log` and 20 minutes of Slack/email searching turned up a few times, but
far less than I remember, so I dropped it. I wonder if there's another, more
clever thing that I could've looked at.

Absence of evidence is not evidence of absence. Clear-cut features or
user-facing defects have a tendency to be favoured in the evidence generation
process. Nobody is out there measuring what the Engineers waste their time on,
and it's a very hard problem anyway, similar to automatically measuring
"performance" which thankfully I have never been subjected to.
