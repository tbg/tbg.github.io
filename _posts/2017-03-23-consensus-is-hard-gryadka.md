---
layout: post
title: "Gryadka is not Paxos, so it's probably wrong"
comments: true
permalink: "if-its-not-paxos-its-probably-wrong-gryadka"
---

*[Leslie Lamport][lamport] purportedly once said (though I can't prove it):*

> If it's not Paxos, it's probably wrong.

*In this post, I take a look at [gryadka][gryadka], which claims to be built on
top of a Paxos-backed CAS register, and conclude that the register is neither
Paxos-backed nor a correct CAS register, serving as a gentle reminder that
consensus is hard.*

---

**Update (3/28/17): an earlier version of the article simplified the
counter-example too much, and left some readers thinking that it wasn't one.
The exposition has been updated; the [repository](https://github.com/tschottdorf/tschottdorf.github.io/) retains the original.**

In a [recent post][rystsov-consensus], [@rystsov][rystsov-tw] explored
alternatives to the distributed consensus seeker's standard choices,
[Multi-Paxos][mp] or [Raft][raft], and presented his proof-of-concept
key-value store named [gryadka][gryadka].

The main ingredient is from an [earlier post][rystsov-cas] in which the claim
is made that you can use Paxos to implement a CAS (compare-and-swap) register,
but "without the log". This suggests that somehow a single instance of Paxos is
used to decide on more than one value. I got suspicious and took a closer look.

## Single Decree Paxos: Delivers exactly what's on the tin, not more

> "Long story short, we hear a story too good to be true... it ain't."
>
> -- Lt. Aldo Raine, Inglorious Basterds

Ideally, for this section you know roughly how single-decree Paxos works. If
you don't, don't worry - but you can read about it in [this simple
paper][paxossimple] (unless you want to [torture yourself with the original
one][paxos-obscure]; please don't).

Either way, you should know that one fundamental truth about single-decree
Paxos is that it decides only on a single value and that changing anything
about that means a) you're no more using Paxos, and more importantly, b) that
it's probably wrong.

Suspecting that both a) and b) were true in this case, I took a look at [the
source][gryadka] and indeed it looks like each acceptor only keeps state on one
instance of Paxos and that this state is updated in-place in a way that looks
superficially like Paxos, but is more reminiscent of some version of a much
weaker algorithm, the [ABD algorithm][abd], with some modifications to try to
turn it into something it's not, namely compare-and-swap.

In vanilla Paxos, when you discover an existing value during `PREPARE`, you
have to proliferate that value. Quoting directly from [Paxos Made
Simple][paxossimplearticle] (emphasis mine):

> If the proposer receives a response to its prepare requests [...] from
> a majority of acceptors, then it sends an accept request to each of those
> acceptors for a proposal numbered n with a value v, where **v is the value of
> the highest-numbered proposal among the responses**, or is any value if the
> responses reported no proposals.

So, always, Paxos will *force you* to use the value that's already there.

The key protocol change in [gryadka] is that you get to *transform* that value,
pretty much arbitrarily, before trying to get the outcome of your
transformation accepted on a majority. Of course, the model mutation of
relevance here is compare-and-swap:

```python
def cas(old, new):
  def change(val_from_prepare):
    if val_from_prepare!=old: raise Conflict(x)
    return new
  return change
```

The reason Paxos forces you to use the existing value is that it forces you to
participate in the commit of said value. When you receive the value from the
`PREPARE` phase, it may not have been committed yet - after all, you got it
sent from a random acceptor, perhaps the only one aware of it. But you are then
forced to go and to spread that value around, and are allowed to take it as
committed only when you have made sure it really is. And since nobody who sees
the value is allowed to spread anything else, consensus is reached.
Of course, at that point the instance becomes worthless for anything but finding
out what the value is - you've got a distributed write-once register.

What gryadka does - smuggling a very powerful operation, namely being able to
transform the value more or less arbitrarily before you spread it - is wrong.
That value you base your mutation on may *not* be committed at the time of your
transformation, but the result you are disseminating can be, though it is not
grounded in any value that was ever visible.

To put it bluntly, you're at risk of using the result of a dirty read as the
basis of a committable mutation.

Now let's see this in (theoretical) action and defer a [discussion of the
proposal numbers involved](#proposal-numbers) to the end of this section.

Take this perfectly reasonable history on three acceptors `A`, `B`, and `C,`
using a single proposer. We'll keep track of the state in the form

```
// Acceptor A has accepted x at ballot number 1, and
// has promised not to accept ballots numbers < 5.
A: (value=x  ballot=1  promised=5)
```

We start from the initial state below. Our proposer will want to run
a compare-and-swap from `asd` to `foo`, denoted `CAS(asd->foo)`.

```
A: (value=asd  ballot=0  promised=0)
B: (value=asd  ballot=0  promised=0)
C: (value=asd  ballot=0  promised=0)
```

Getting ready, our proposer runs a prepare phase at ballot 1. It learns the
value `asd`.

```
A: (value=asd  ballot=0  promised=1)
B: (value=asd  ballot=0  promised=1)
C: (value=asd  ballot=0  promised=1)
```

`asd` matches the expected value of the intended command `CAS(asd->foo)` at
ballot 1, so it will try to get the transformed value `foo` accepted. The
proposer gets `A` to accept the value `foo`, but then gives up (asynchronous
network!), leaving the system in the state

```
A: (value=foo  ballot=1  promised=1)
B: (value=asd  ballot=0  promised=1)
C: (value=asd  ballot=0  promised=1)
```

Now let's query the register from different (throw-away, i.e. only used this
one time) proposer for ballot number 2. Networks being asynchronous, we don't
manage to talk to `A`, but the majority `(B, C)` works.

A read is vanilla Paxos, i.e. a round of `PREPARE` to get the highest accepted
value known to our majority, and then an `ACCEPT` phase with that value to the
majority.

Of course, the result will be `asd`; the register thus certifiably still stores
the value `asd` and we have the state

```
A: (value=foo  ballot=1  promised=1)
B: (value=asd  ballot=0  promised=2)
C: (value=asd  ballot=0  promised=2)
```

Next, our proposer prepares for a `CAS(foo->bar)` at a higher ballot, say (for
simplicity) `3`.

After `PREPARE`, we have

```
A: (value=foo  ballot=1  promised=3)
B: (value=asd  ballot=0  promised=3)
C: (value=asd  ballot=0  promised=3)
```
The highest returned accepted value will be `foo` from `A`, and this matches
the expectation of the `CAS`. Uh oh, shady! `foo` isn't actually committed, and
consequently the value the proposer is going to get committed is not grounded
in the "real" value of the register, `asd`.

But misfortune runs its course, and `B` gets everyone to accept the value
`bar`, wiping out the "dirty" value `foo` in the process:

```
A: (value=bar  ballot=3  promised=3)
B: (value=bar  ballot=3  promised=3)
C: (value=bar  ballot=3  promised=3)
```

You have successfully managed to read the value `asd` and then compare-and-swap
from `foo` to `bar`. Oops!

You can see that the problem immediately goes away when you don't allow
transforming the discovered prepared value before you try to go to the `ACCEPT`
phase because that forces everyone use the same value, whereas now each
proposal basically gets to invent one. In other words, Paxos is correct, but
the above algorithm is not Paxos and, unfortunately, incorrect.

To restore sanity, you must disallow mutations except for the identity:

```python
def ident(maybe_value):
  def change(val_from_prepare):
    if val_from_prepare!=empty:
        return val_from_prepare # spread earlier value
    return maybe_value          # we get to choose
  return change
```

Doing so collapses [gryadka][gryadka]'s algorithm back into single-decree
Paxos, and the resulting data structure is the associated write-once register.

<span id="proposal-numbers"></span>
Follow the actual source code of [gryadka][gryadka] you'll
notice that you have to work harder to get a "real" counterexample, and it's
really not surprising to me that this anomaly wasn't found during testing:

- Reading values is vanilla Paxos and so pretty efficiently wipes out any
  "partial anomaly" unless you have just the right network partition in place,
  as we did in the example.
- Any leadership change (which is usually guaranteed when working with multiple
  proposers) invalidates all read values and so triggers a lot of reads, again
  patching things up.

A practical version of this example would have to "prime" the proposals so that
the ballot numbers work out. Ballot numbers have the form

```(term, proposer_id, counter)```

and are compared lexicographically.

- `term` is learned from responses; when you try to be come the leader, you
  pick the discovered `term` and increment it. So it's easy to let two
  proposers end up in the same term.
- `proposer_id` is fixed per proposer; you get to choose that as well.
- `counter` increases with any attempt at a proposal (even if they never make
  it anywhere else); hence this component you basically get to choose however
  you like.

Priming the ballot numbers is hence doable, if a bit awkward, and you could
replace the "symbolic" ballot numbers in the example using well-ordered
`proposer_id`s and making everyone use the same `term` (`count` doesn't matter):

- `1` becomes `(1, 1)` (the failed CAS, now from a distinct proposer)
- `2` becomes `(1, 2)` (the read of the old value)
- `3` becomes `(1, 3)` (the "successful" CAS).

There is more to worry about like the version numbers built into gryadka, but
they actually open up more anomalies if you follow the above scheme, like
version `n` being able to replace version `n`. All you need is an
uninterrupted sequence of `CAS`.

And last but not least, you could take a closer look at the leadership
mechanism: you can likely cook up more havoc using the caching of read values
(provoking version number problems).

Hopefully it's clear at this point that prepare-modify-accept is fundamentally
flawed and its shortcomings can be masked and made more unlikely but not
patched.

## Theoretical discussion

You may have liked what [gryadka][gryadka]'s algorithm would have given you,
and wonder now whether Multi-Paxos is the only useful and/or sane abstraction
on top of single-decree Paxos. Is it possible to get compare-and-swap without
the log?

*TL;DR: I don't think so.*

If you look at one of the classics in the field, [Herlihy's
Wait-Free Synchronization paper][herlihy], which talks about problems similar
to this one, defines the *Consensus Number* as

> the maximum number of processes for which the object can solve a simple
> consensus problem.

and proves that

> In a system of n or more concurrent processes, [...] it is impossible to
> construct a wait-free implementation of an object with consensus number
> n from an object with a lower consensus number.

For example, atomic multi-reader multi-writer registers have consensus number
1, which is a consolation prize - you can't get far if that's all you use. On
the other hand, a compare-and-swappable register has consensus number ∞, so
it's a universal object in this hierarchy.

Next, the only universal object Herlihy explicitly constructs is, basically,
Multi-Paxos (see [MIT/EECS/Fall09#23][mit]), and additional work on the
robustness of this hierarchy suggests that the

> consensus number appears to give a real classification that allows us to say
> for example that any collection of read-write registers (consensus number 1),
> fetch-and-increments (2), test-and-set bits (2), and queues (2) is not enough
> to build a compare-and-swap (∞).
>
> -- [Yale CS465/565][yalecs]

This seems to indicate that you really do need an infinite number of
"instances", and that it should be damn near impossible to improve on
Multi-Paxos. But, I for one, would be delighted to be proven wrong.

### Multi-Paxos

I'll end this post with a short section on [Multi-Paxos][mp], which has
a reputation for being hard to understand. It's almost fashionable to spread
that rumor around.

I don't think it's that hard to understand. It's certainly hard to *implement*,
but the basic idea is simple, obviously correct, and doesn't require anything
but a decent understanding of (or trust in the correctness of) single-decree
Paxos, which you can [easily achieve][paxossimple].

Multi-Paxos is nothing but a log represented by a growing sequence `I` of Paxos
instances, where `I[0]` is the instance that determines the first log entry,
`I[1]` the second, and so on, with instances created as they are needed, and
disposed of when the state machine has consumed their chosen value.

With that set up, you take your state machine and let it work through the log,
that is, once `I[0]` is determined, you apply it to the state machine and wait
until `I[1]` also materializes, at which point you may apply that as well.

That settles everything except for how commands actually get proposed, which is
where [Multi-Paxos][mp] reaps its important performance benefits: under normal
operation, a single proposer (the soon-to-be "leader") proposes all commands as
follows:

1. it creates a ballot number and sends a `PREPARE` message that is received by
   all instantiated, undecided groups (note that it won't make a difference for
   instances that have already been decided), and will be automatically copied
   to all new groups created until another leader takes over. Or, put
   differently, you're running this `PREPARE` for all past and future instances
   simultaneously, receiving responses only from the finite number of groups
   that are instantiated but undecided.
1. it fills any gaps in the log (by spreading the existing value, if any, or
   submitting a no-op otherwise)
1. it is now free to send `ACCEPT`s to all "new" instances repeatedly, hence
   deciding new commands, until another proposer interferes by running its own
   `PREPARE` (thus invalidating all of "our" `PREPARE`s sent initially) and
   taking over.

The details get more involved (leading into the jungle of implementation), but
note how the basic idea is obviously correct, because we didn't mess with Paxos
at all but simply saved a few roundtrips through multiplexing the initial
`PREPARE` message to the infinity of instances.

[lamport]: http://lamport.azurewebsites.net
[rystsov-cas]: http://rystsov.info/2015/09/16/how-paxos-works.html
[rystsov-consensus]: http://rystsov.info/2017/02/15/simple-consensus.html
[rystsov-tw]: https://twitter.com/rystsov
[mp]: https://en.wikipedia.org/wiki/Paxos_(computer_science)#Multi-Paxos
[raft]: https://raft.github.io/
[paxos]: https://en.wikipedia.org/wiki/Paxos_(computer_science)#Basic_Paxos
[gryadka]: https://github.com/gryadka/js
[paxossimple]: https://www.idi.ntnu.no/emner/tdt02/PaxosMadeSimple.pdf
[paxossimplearticle]: https://www.microsoft.com/en-us/research/publication/paxos-made-simple/
[paxos-obscure]: http://lamport.azurewebsites.net/pubs/lamport-paxos.pdf
[abd]: http://dl.acm.org/ft_gateway.cfm?id=2500874&type=pdf&path=%2F2510000%2F2500874%2Fsupp%2Fp88-musial-supp.pdf&supp=1&dwn=1
[herlihy]: https://cs.brown.edu/~mph/Herlihy91/p124-herlihy.pdf
[yalecs]: http://www.cs.yale.edu/homes/aspnes/classes/465/notes.pdf
[mit]: http://courses.csail.mit.edu/6.852/08/lectures/Lecture19.pdf
