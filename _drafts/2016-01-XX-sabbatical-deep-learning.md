---
layout: post
title: "Sabbatical Reading List: The Deep Learning Book"
comments: true
permalink: "sabbatical-deep-learning-book"
---

2016 has been a hip year for Artificial Intelligence and, by extension, for
Deep Learning. What Deep Blue did to chess in XXXX is now being done to Go,
image-recognizing deep neural networks can dream up bizarre works of art, and
now that Stephen Hawking is "concerned" about AIs taking over the world, SkyNet
seems like a done deal.

At that point, it's a good idea to pick up the (as of January 2017) newly
released seminal [Deep Learning Book](http://deeplearningbook.org) and get
a grip on reality, and, as a side effect, on a thriving subject in the applied
computer sciences.

Don't get me wrong, Deep Learning is a hugely interesting subject with loads of
progress. But, at heart, the credit for the success of it goes to clever humans
who have found good models for the specific problems they were trying to solve
- whether it's image recognition, Go, speech transcription or any other of the
myriad of applications.

At its core and simplifying slightly, Deep Learning is about
* neural networks with hidden layers, which roughly corresponds to
* approximating a continuous function (or probability density) by composing
  simpler functions,
* minimizing a cost function, but having to eyeball it most of the time because
* only test data is known (if even that), having to do loads of linear algebra
* efficiently, and dealing with overfitting, underfitting, numerical stability,
* bugs, ...

To me, it certainly seems that the successes of these techniques can be
credited to the brains that thought they were a good idea to try and address
the problem at hand, and a subsequent eternity spent programming and, unless
superhuman, fixing bugs.

Luckily, in the day and age of free datasets, XXXX, XXXX and TensorFlow we all
can play around with at least the more basic deep neural networks. What can't
be automated is the modeling expertise mentioned above - something a human
brain has to do!

To that end, the Deep Learning Book walks you through an introductory chapter
on off-the-shelf machine learning subjects, linear algebra and probability and
then wastes no time before moving to a practical introduction to feedforward
and recurrent networks. Chapters are short and dense enough to be digestible,
but (I think) contain enough information to look at and grok an actual
implementation of a neural network one would find in the wild, ask informed
questions, or go off and read implementation papers.

A third section covers the academical (as in research) side. I haven't had the
chance to dig into it yet, but it clearly attempts to show you the ropes for
digging into the latest research papers.

At time of writing, the [Deep Learning Book](http://deeplearningbook.org) is
available for free on-line or as a (paid) hardcover. Perplexingly, it's not
available as an ebook, or I would have paid for the added comfort of not having
to cache some chapters before going offline, and perhaps better rendering on
mobile devices.

There's a fledgling exercise section, the substance of which is mostly a link
to [TensorFlow's tutorial section](#TODO), which is worth a look.

Relatedly but not associated with the book, I also yet have to check out
[fast.ai's seven-week practical online
course](http://course.fast.ai/index.html) which sounds promising in that you
get to tinker and play with all the boiler-plate removed and educational
content woven in.
