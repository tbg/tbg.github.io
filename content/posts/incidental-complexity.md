---
title: "Pricing Incidental Complexity"
date: 2023-01-13T08:02:40+01:00
draft: true
---

Every Software Engineer knows that uncontained complexity is expensive: employee time is scarce and spending it, perhaps even repeatedly, on circumnavigating poorly architected areas of the codebase, is unfortunate.

I spent the entirety of last morning dealing with the fallout of incorrectly handling a piece of largely unnecessary complexity in the CockroachDB code base. I was familiar with the complexity; I was around when the complexity was introduced many years ago; I field questions (by others working on CockroachDB) about it a few times a year; I was, yesterday morning, working specifically in an area touched by the complexity and in fact had improved its code documentation. And yet, it got me and I wasted a few hours, and not for the first time either.

This made me want to at least try to quantify the cost of this particular misshapen piece of (largely accidental) design. I tried searching the git log for patches that touched the concept:

```
git log --pickaxe-regex -S '\\x02|LocalMax'
```

and did something similar in Slack. It did turn up some others struggling with the same but I think it missed most things and even for the hits returned a lot of noise. Issues with Slack search notwithstanding, I remember a number of conversations over the years that I can't find with that method.
