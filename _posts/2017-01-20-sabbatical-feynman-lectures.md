---
layout: post
title: "Sabbatical Reading List: The Feynman Lectures"
comments: true
permalink: "sabbatical-feynman-lectures"
---

**I'm on sabbatical, so I get to look into things without the distractions of
my day-to-day job. This is one of them.**

Feynman was not a saint[^not-saint], but without doubt one of the geniuses of the 20th
century. And I am, without doubt, someone who in their confused highschool
years took the *genius* decision to opt out of physics once that was
possible, presumably to spend more time staring at a computer screen).

Three months ago, with half a year of sabbatical at my disposal, I found myself
picking up (i.e. navigating my browser to) the freely available *Feynman
Lectures on Physics*[^lectures] and have since had trouble putting
them down.

Feynman was sometimes referred to as "The Great Explainer", referring to his
uncanny ability to decompose and boil down the complicated until it looked
simple and digestible, for leaving no stone unturned, and for playfully
handholding his audience to a deep understanding of the matter at hand, one
toy example at a time.

Hopping on Feynman's bandwagon works well only to some extent, though. What you
get are lucid explanations and great intuition about physical processes, with
lots of little examples to *really* make you understand how it works.

What you don't get are solid proofs. You could say, who needs a proof when
intuition makes it all clear as day? And I, with a doctorate in a heavily
physics-influenced part of pure mathematics[^pde], would agree and trust myself to be
able to spot sketchy arguments.

But - in volume two of the same Feynman Lectures, Feynman got the Faraday
cage[^cage] shielding effect completely wrong: He
predicts shielding *exponential* in the mesh gap, while it's in fact
*linear*[^mistake].
The mistake is assuming that the wires (discs in 2D) of the cage can be made
infinitely thin. But going from constant-voltage discs to point charges changes
the question you're answering.

That is one thing, but we all make mistakes. The other caveat is that once you
get into the quantum world, you will have to face the fact that Feynman was
fundamentally a "particles guy", which is problematic because particles are
just a sometimes convenient way of talking about fields which are highly
localized, but not a fundamental concept.

Assuming the particle perspective when that doesn't apply brings up apparently
paradox situations such as that observed in the double-slit experiment, which
isn't a paradox when you realize you're observing *fields*. In short, *there
are no particles, there are only fields*[^only-fields], and
Feynman appears to be on the wrong side of teaching history (while certainly
having been on the right one scientifically). At least, as the
article[^only-fields] 
points out, most of teaching is on the wrong side of history on that point,
presumably due to the fact that particles come naturally to humans and even
more so when you enter the quantum world through classical Newtonian mechanics.

There also aren't any exercises in the original lectures, though a list is
maintained at CalTech, along with errata and notes[^lectures]. For more
complete coverage, a book exists[^exercises], but reviews point out that many
problems are presented without a solution, which can be off-putting.

With that in mind, I have so far passed many hours reading my way towards the
end of volume one of the lectures and, so far, I can strongly recommend them.
Imagining myself in my first semester in this class (as it was originally
taught) would've surely steamrolled me.

But maybe that's what it means to study physics?

[^not-saint]: [The problem of Richard Feynman](https://galileospendulum.org/2014/07/13/the-problem-of-richard-feynman/).
[^cage]: [Faraday cage (Wikipedia)](https://en.wikipedia.org/wiki/Faraday_cage)
[^mistake]: [Surprises of the Faraday Cage](https://sinews.siam.org/Details-Page/surprises-of-the-faraday-cage), Lloyd N. Trefethen, 2016.
[^only-fields]: [There are no particles, there are only fields](https://arxiv.org/abs/1204.4616), Art Hobson, 2012.
[^lectures]: [feynmanlectures.info](http://www.feynmanlectures.info/)
[^exercises]: [Exercises for the Feynman Lectures on Physics](https://www.amazon.com/Exercises-Feynman-Lectures-Physics-Richard/dp/0465060714) on Amazon.com.
[^pde]: Partial Differential Equations, more precisely the analysis of nonlinear dispersive equations, and even more precisely, a result on some [quadratic Klein-Gordon equations](https://arxiv.org/abs/1209.1518).
