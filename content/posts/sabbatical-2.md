---
title: "Sabbatical Dispatches, September 2023"
date: 2023-10-02T23:22:11-07:00
draft: false
---

## Travel Updates

We spent close to two weeks in the Bay Area, spending time with colleagues, relatives, old friends, and new friends (
thanks Burning Man). We stayed right in the middle of Alameda, which turned out to be perfect - quiet neighborhoods full
of beautiful gardens, parks, playground, and coffee in walking distance, and the O bus to SF right by our doorstep. We
basically got to pretend we were locals for two weeks, and I got my first ever ride in a fully-self driving car (a Waymo
cab). The wildfire smoke was a small damper and seems to be regular
enough now that it would factor into my decision process if we were to entertain moving out West (we are not).

We're now in Seattle (think warm Fall weather, leaves turning, and pumpkin harvest), where we're staying for just over a
week, with a former colleague turned family friends of ours.
Our kids say they'd like to live here, so I can confidently say they're having a good time, and so are we. There's a
CrossFit gym across the street which is amazing (I learned that not only do I like regular exercise, I *need* it to be
able to feel truly happy). And for the rare day during which we don't get to hang out with a
colleague or friend, Seattle has a lot to offer.

Next week we are heading out to Yellowstone NP. A week after that, likely down to Kanab (South Utah) for another
week-ish. Next, LA and a flight out to Costa Rica from there.

## Links

- [The SF Bay is only ~10k years old](https://education.savingthebay.org/wp-content/guides/The-Formation-of-San-Francisco-Bay.pdf) (the coastline was 27m west, past the Farallon islands, during the last Ice Age)
- [How to lose weight in four easy steps](https://medium.com/@AaronBleyaert/how-to-lose-weight-in-4-easy-steps-1f135f7e1dec). Don't be fooled by the title, it's great advice and beats [the Kaua'i diet](https://tbg.github.io/posts/kauai-diet/). Also available in [video form](https://www.youtube.com/watch?v=S2yTZNCEK1w).
- [Tradle](https://oec.world/en/tradle/). I find it quite hard.
- [MMAcevedo](https://qntm.org/mmacevedo), a great piece of Black Mirror-esque writing that I return to often.
- [This Is Water](https://fs.blog/david-foster-wallace-this-is-water/): I thought everyone knew this but apparently not so posting just in case.

### Voice Input

Since I am on my Android phone most of the time now and realized that typing on a phone essentially preempted lots of
communication, I gave voice input another shot. It works much better than I remember, especially after
following [these tips](https://support.google.com/gboard/answer/11197787?hl=en). Unfortunately, sometimes it does go off
the rails and *editing* the text is really really [annoying](https://jenson.org/text/).

I wish the words and the resulting text were kept connected after the fact so that you could select from alternative
suggestions or use voice-input hints.

On a related note, I also wish WhatsApp voice messages came with a transcription that has clickable words with a similar
feature (play the snippet of audio corresponding to that transcribed word). It'll probably come once the models are
cheap
enough for commodity mobile hardware.

### Bone Conducting Headphones

Despite going through numerous earbuds over the year, I never found one that would reliably stay in during a workout. I
got the [Shokz Openrun](https://shokz.com/products/openrun) last week. Still need to use them more to form an opinion,
but they are quiet. Walking on a noisy street, it's hard to really hear your podcast. They're not great for bass-heavy
music (this much I knew before and accepted). They have a proprietary charger, which is sad (but all the alternatives I
looked at also had one). The microphone doesn't seem to be working too well, according to my conversation partners so
far. But, no worrying about losing an earbud, so maybe they will become my new go-to anyway.

### Brilliant

I got a subscription to use my brain whenever I get a chance. The Introduction to Neural Networks course[^inn] is very
well done: it manages to get all of the concepts across in an interactive way and is really a pleasure to go through
even if you basically "already know everything". Whoever worked on this, great job.

[^inn]: https://brilliant.org/courses/intro-neural-networks/

## Work-related stuff

I finished Isaacson's new Musk biography. I had read the Vance one before but still worth reading. Relatedly, Nvidia's
Jensen Huang is an interesting case[^jh].

Keith Rabois on [How To Operate](https://m.youtube.com/watch?si=2Fu1Uylek1hzAfbC&v=6fQHLK1aIBs&feature=youtu.be#bottom-sheet) was fun (thanks to AA for the pointer).

[CockroachDB trivia](https://twitter.com/eatonphil/status/1708580865951879512)

https://twitter.com/eatonphil/status/1708580865951879512

[^jh]: https://twitter.com/danhockenmaier/status/1701608618087571787

Thankfully I am on a hiatus from looking at JSON, but once I'm back in the rat race I'll give [fx](https://fx.wtf/) a
shot and thanks to writing this here, I'll remember to actually do it.

### Some Quotes to Live By

> For each desired change, make the change easy, then make the easy change

(Kent Beck[^kb])

[^kb]: https://twitter.com/KentBeck/status/250733358307500032?lang=en

> Everyone knows that debugging is twice as hard as writing a program in the first place. So if you're as clever as you
> can be when you write it, how will you ever debug it?

(Brian Kernighan[^bk])

[^bk]: https://www.goodreads.com/quotes/273375-everyone-knows-that-debugging-is-twice-as-hard-as-writing

[Always Measure One Level Deeper](https://dl.acm.org/doi/10.1145/3213770) (John Ousterhout)

### Deterministic Cockroaches 

Some projects, notably FoundationDB and Tigerbeetle, have made a large upfront investment in determinism. If your
executions can be made deterministic. Deterministic software is much easier to investigate[^1].
In CockroachDB, we have not done this and I spend a fair amount of time (for someone on a sabbatical, anyway) mulling
whether this was a one-way door decision[^2].

For deterministic simulation testing, you need deterministic code, deterministic network and other I/O, deterministic
scheduling, etc., which means that it is difficult to "just" retrofit. Projects like Antithesis[^3] exist but aren't
as easy to work with.

I don't like everything about how `etcd-io/raft`[^4] is architected, but it's nice and deterministic. All concurrency
and
I/O are outside of the purview of the library. As a result, it was easy to add a datadriven test harness a few years
back[^4]. I am hoping, and have been advocating for [^5] extending that pattern at least into the lower layers of
CockroachDB. But at some point, you have to take the plunge and start aggressively changing the architecture of the
codebase. A glimpse of what that could look like is in this tweet[^6] by Dominik Tornow. It's quite invasive, has an
inner-platform effect[^7] feel and I just
can't imagine it ever happening on CockroachDB's codebase.

At the same time, if I got to do a redo on CockroachDB, I'd still do whatever it takes to get fast deterministic
simulation testing.

[^1]: https://db.cs.cmu.edu/events/databases-2022-tigerbeetle-magical-memory-tour-joran-dirk-greef/
[^2]: https://aws.amazon.com/executive-insights/content/how-amazon-defines-and-operationalizes-a-day-1-culture/
[^3]: https://www.antithesis.com/
[^4]: https://github.com/etcd-io/raft/blob/645ea1204eae232c11df11e47ab1bc7ff7240ea1/testdata/replicate_pause.txt
[^5]: https://github.com/cockroachdb/cockroach/issues/105177
[^6]: https://twitter.com/DominikTornow/status/1670098981148405760
[^7]: https://en.m.wikipedia.org/wiki/Inner-platform_effect