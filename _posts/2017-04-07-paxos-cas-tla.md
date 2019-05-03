---
layout: post
title: "Paxos on Steroids and a Crash Course in TLA+"
comments: true
permalink: "single-decree-paxos-tla-compare-and-swap"
---

[@rystsov][rystsov-tw] recently posted an interesting
[article][rystsov-consensus] in which he describes a peculiar generalization of
[Single-Decree Paxos][paxossimple] that could, in some situations, replace
[Multi-Paxos][mp]. I was skeptical: this looked really useful; why wasn't
everyone aware of it already?

I still don't know the answer to that question, but I've since failed at
constructing a counter-example, and find that the algorithm *looks* "correct"
(more on what that means later).

That's exciting! Today, embark with me on what will hopefully become a series
of posts on what (at least here) will be called *Compare-And-Swap Paxos*
(CAS-Paxos), and how to explore its properties through formal specification.

A reasonable first step is a tutorial-style formal description of the algorithm
in [TLA+][tla], which among other things is also a model checker which allows
you to specify the algorithm and explore concrete histories it permits.

This then, is the plan for today:

1. [Quick Review of Linearizability](#review-linearizability)
1. [Quick review of Single-Decree Paxos](#review-single-decree-paxos)
1. [(Colloquial) Description of CAS-Paxos](#verbal-description-of-cas-paxos)
1. [Tutorial: Specifying CAS-Paxos in TLA+](#specifying-cas-paxos-in-tla)
1. [Tutorial: Discovering Interesting Histories](#discovering-interesting-histories)
1. [Discussion: CAS-Paxos and its surprises](#discussion-of-cas-paxos)

## Review: Linearizability

We are talking about registers today (i.e. an abstract system that, no matter the
implementation, exposes and operates on a single value), so for the purpose of
making everything hands-on, imagine your system to be a distributed key-value
store with only a single active key (perhaps [CockroachDB][crdb] or one using the
algorithm described later), and that you have multiple clients running
compare-and-swaps on that key.

Linearizability of that system (i.e. all the clients writing to that key) holds
if no matter how things interleave, for each operation in any possible history, you can find an instant at which it (and only it)
applied atomically, and that instant is between the moment in time at which
the respective command was initiated (i.e. a request dispatched by a client)
and acknowledged (i.e. a response received by the client).

For example, if the register is initially `foo`, and at

- `time=10`: I dispatch `CAS(bar, baz)`
- `time=15`: You dispatch `CAS(foo, bar)`
- `time=20`: I receive `OK` from the system
- `time=30`: You receive `OK` from the system

then that history is linearizable: my CAS succeeded, so your operation must
have applied to the system before mine; otherwise the register wouldn't have
had the value `bar` I'd assumed I'd seen. And that is well possible because you
initiated your CAS before the system told me mine had applied, and so we can
arrange them atomically in a history for the system:

- `time=10`: I dispatch `CAS(bar, baz)`
- `time=15`: You dispatch `CAS(foo, bar)`
- **`time=16`: Server executes your `CAS(foo, bar)`**
- **`time=17`: Server executes my `CAS(bar, baz)`**
- `time=20`: I receive `OK` from the system
- `time=30`: You receive `OK` from the system

[Kyle Kingsbury (aphyr) does a much better job][aphyr-lin] describing this, so you should go
there unless you're already comfortable with the concept.

## Review: Single-Decree Paxos

[Single-Decree Paxos][paxossimple] is a consensus algorithm that lets
a number of so-called acceptors agree on a single value (out of many that may
be offered to them) and preserve that value for eternity, unless a quorum of
acceptors holding on to a copy of the value perish, never to be seen again.

To clients, the abstraction provided is a linearizable register that can be written exactly
once and, once written, exposes the same value forever.

There are good resources for learning about Paxos [here][paxossimple] (slides
with examples!) and [here][paxossimplearticle] (words only), so we only review
it briefly. You absolutely want to make sure that you understand simple Paxos
well before you continue.

Paxos is organized around ballot (or proposal) numbers, and the goal is
generally to get successful responses from a majority of the acceptors. Clients
("proposers") which want to propose a (or learn the, if any) value pick
a unique proposal number (in practice, one that they think might actually
succeed) and try to take it first through a `PREPARE` phase and, if a majority
of acceptors indicates success, an `ACCEPT` phase.

At the end of a successful `PREPARE` phase, the proposer has learned from
a majority of the acceptors

- that they promise to not service any smaller ballots than the proposer's, and
- whether there is already a value (not necessarily committed) which must be picked
  up or whether the proposer gets to choose one.

If any values are returned from `PREPARE`, the proposer must use the value
corresponding to the highest ballot (and there are never two values for any
ballot).

The proposer can then proceed to the `ACCEPT` phase, in which it tries to store
the chosen value on all acceptors (but needs only a quorum); an acceptor will
refuse values for a ballot that it has promised not to service any more.

Unsuccessful proposals are usually caused by network issues, the proposer
crashing half-way through, or (most interestingly) multiple proposals
overlapping concurrently. But, no matter the interleaving, after the first
successful `ACCEPT` phase, the value is set in stone (barring failure of
a majority of acceptors) and all successful proposals will result in the
same value - a register backed by Single Decree Paxos is linearizable.

## Verbal Description of CAS-Paxos

CAS-Paxos is derived from Single Decree Paxos with just one small but impactful
change: After the `PREPARE` phase, the proposer is allowed to pick a new value
(perhaps taking into account the value, if any, learned from the `PREPARE`
phase).

For the purposes of this discussion, we'll settle on that all proposers want to
run compare-and-swaps: if the value they learn is the one they expect, they go
ahead and try to get a new value accepted. To simply read the value, you run an
instance of Single-Decree Paxos instead.

It's not clear at this point of the posting, but the resulting algorithm
actually seems to make basic sense (as in, it provides guarantees which are
useful), though we'll discover a few caveats. And, though we won't even get
close to proving it today (and I haven't proved it), the semantics of the
register, once suitably defined, should be linearizable!

For now, the bird-eye view: whereas before we were stuck with a write-once
register, now we seem to have something on our hands that can change value as
often as we'd like, and perhaps could serve as the building block of
a key-value store.

Let's discuss the concrete behavior of the algorithm later and do some TLA+
first.

## Specifying CAS-Paxos in TLA+

Don't care about writing the spec? [Jump straight into the
analysis](#running-the-model).

Don't try to copy-paste from these examples; they've been formatted to fit the
screen. Instead, to follow along, head on over to the [caspaxos-tla
repo][caspaxos-tla]; the initial commits on the master branch track this post.

A TLA specification starts with a list of module imports. We'll deal with
integers and will want to talk about the cardinality of sets, so we import

```
EXTENDS Integers, FiniteSets
```

From what I can tell, in practice you use whichever operators you find on the
internet and when it fails, realize that there is something you needed to
import, which you then add.

### Constants, Basic Definitions, and Assertions

When you specify a model, you leave certain parts configurable so that you can
invoke the model checker with various settings.

For CAS-Paxos, we have a set of possible values the register can take, a set of
oblique acceptors for which we really only care about how many they are, and
a mutator which maps a ballot number and a value to a new value.

```
CONSTANT Values, Acceptors, Mutator(_, _)
```

Throughout this whole exercise, keep in mind that the state space which needs
to be explored by the model checker exponentially spirals out of control. For
example, you certainly can't hope to check all possible histories when the
register can hold 100 different values (in fact, 10 is probably out of reach
already); I'm using 4 throughout this example simply because that finishes in 15
seconds on my machine. Note also that we don't restrict the ballot numbers
just yet; that's achieved later, by overriding the definition of the natural
numbers while checking the model.

`Mutator` is a good approximation of how compare-and-swap'ping proposers would
choose the new values (Single-Decree Paxos corresponds to `Mutator(ballot,
foundValue) = foundValue`), abstracting away nondeterminism by using the ballot
number to decide on the new value. You could let the model checker generate
"real" CAS operations as well, but let's agree that this is interesting enough,
and that it's easy enough to extend this later.

Now, we define the set of all quorums. The crucial property that we need is
that any two quorums intersect nontrivially, and the `ASSUME` command lets the
model checker verify that it holds in the concrete instances we run it for.

Since we're lazy, we use the set of all majorities of acceptors as the set of
quorums. This is what you usually see in practice because it maximizes the
number of acceptors you can afford to lose.

```
Quorums == {
  S \in SUBSET(Acceptors) :
    Cardinality(S) > Cardinality(Acceptors \ S) }

ASSUME QuorumAssumption == /\ \A Q \in Quorums : Q \subseteq Acceptors
                           /\ \A Q1, Q2 \in Quorums : Q1 \cap Q2 # {}
```

What this does is express that

- `Quorums` is the set of those sets of acceptors for which more acceptors are in
  the set than not. That's just a fancy way of collecting all subsets 
  containing more than half of everyone.
- `QuorumAssumption` has to be true or the model checker aborts. It is
  satisfied if:
  - all the quorums we declared actually consist of acceptors (something that
    you probably implicitly assumed anyway), and
  - any two sets of quorums have nonempty intersection.

On to ballot numbers, for which we define

```
Ballot == Nat
```

This just means that we define a set `Ballots` which equals the natural numbers
0, 1, …. As mentioned earlier, when we run the model, we're going to override
`Nat` by some finite subset, typically restricting it to `1, 2, 3` - that's
tiny, but enough to find an interesting history without waiting.

### Messages

Now we define the set of all possible messages. In this specification,
proposers are implicit. Messages originating from them are created "out of thin
air" and not addressed to a specific acceptor. In practice they would be,
though note that each acceptor would receive the same "message body", and
omitting the explicit originator and recipient reduces the state space. Note
also that messages are not explicitly rejected but simply not reacted to. In
particular, the implicit proposer has no notion of which ballot to try next.
The spec lets them try arbitrary ballots instead. Bad for the state space, but
good for generality.

A message is either a prepare request for a ballot, a prepare response, an
accept request for a ballot with a new value, or an accept response.

```
Message ==      [type : {"prepare-req"}, bal : Ballot]
           \cup [
                 type : {"prepare-rsp"}, acc : Acceptors,
                 \* ballot for which promise is given
                 promised : Ballot,
                 \* ballot at which val was accepted
                 accepted : Ballot,
                 val : Values
                ]
           \cup [type : {"accept-req"}, bal : Ballot, newVal : Values]
           \cup [type : {"accept-rsp"}, acc : Acceptors,
                 accepted : Ballot ]
```

Above, `\cup` is the union of two sets, {`"prepare-req"`} is a set containing
as its only element the string `prepare-req`, and `[ x : X, y : Y ]` is the set
of all records (TLAs version of associative arrays) with only keys `x` and `y`
set and their values elements of `X`, `Y`, respectively.

### State of the Model

Since we are modelling only the acceptors and messages, the state of the mode
is only the set of messages in existence, and whatever local state each
acceptor stores:

```
VARIABLE prepared,
         accepted,
         value
VARIABLE msgs
```

The tuple `<<prepared[a], accepted[a], value[a]>>` will be the state that would
be kept on acceptor `a` in a real-world implementation of the spec.

The above says awefully little about what these variables are actually like in
practice, so we add a formula which we later instruct the model checker to
verify in each state, namely that `prepared` and `accepted` map acceptors to
ballot numbers, `value` maps acceptors to values, and that any message sent is
an element of the above set of "legal" messages.

```
TypeOK == /\ prepared \in [ Acceptors -> Ballot ]
          /\ accepted \in [ Acceptors -> Ballot ]
          /\ value \in    [ Acceptors -> Values ]
          /\ msgs \subseteq Message
```

How does one "send" a message, then? Like this:

```
Send(m) == msgs' = msgs \cup {m}
```
TLA models the "next state" with primed variables, so to satisfy the formula
`Send(m)`, the next state's incarnation of `msgs` must contain `m`.

We won't ever remove messages from `msgs`. By modelling `msgs` as a set that
never shrinks, we model that messages may be received multiple times, and that
they can freely reorder. Desirable properties to model because even though
TCP/IP connections don't allow reordering, sometimes we need to reconnect and
no such property holds across connections!

Now, finally, things get interesting and we get to specify the initial state of
the model:

```
Init == /\ prepared = [ a \in Acceptors |-> 0 ]
        /\ accepted = [ a \in Acceptors |-> 0 ]
        /\ value    = [ a \in Acceptors |-> InitialValue ]
        /\ msgs = {}
```

Ok, maybe not too interesting. Note however that the initial state has an
initial committed value, ie the register doesn't start "empty".  This is an
inconsequential simplification.

### Stub Actions

Before we write down the actions that drive the algorithm (i.e. sending and
reacting to messages), we want to finish up the scaffolding. We define mock
actions (actions are just formulas which contain primed variables, meaning that
to satisfy them you must mutate the state):

```
PrepareReq(b) == FALSE
PrepareRsp(a) == FALSE
AcceptReq(b, v) == FALSE
AcceptRsp(a) == FALSE
```

We'll return to those later.

### The Main Formula

Now, finally, we reach the centerpiece of the whole operation: The `Next`
formula, traditionally named such because it is this formula that we'll
instruct the model checker to satisfy by deriving new reachable states.

```
Next == \/ \E b \in Ballot : \/ PrepareReq(b)
                             \/ \E v \in Values : AcceptReq(b, v)
        \/ \E a \in Acceptors : PrepareRsp(a) \/ AcceptRsp(a)
```

Read: Satisfy `Next` by either satisfying `PrepareReq` for some ballot, or
`AcceptReq` for some ballot and value, or by finding an acceptor for which
`PrepareRsp` or `AcceptRsp` can be satisfied.

`Next` is just a name without formal meaning, but `Spec` is the default entry
point for the TLA+ model checker. The below formula is a temporal formula and
means that the valid behaviors of the specification are those which initially
satisfy `Init`, and from each step to the following the formula `Next` is
satisfied, or all the variables stay the same ("stuttering step").

```
Spec == Init /\ [][Next]_<<prepared, accepted, value, msgs>>
```

Now hopefully it is at least somewhat clear how the model checker works: it
constructs all states that satisfy `Init` (there could be many, though here
there's just one), and until it runs out of new states explores all state
transitions that satisfy `Next`. Phew! You can imagine how quickly the state
space blows up as you increase the size of the parameters, and hopefully it's
also clear that any infinite set is likely going to lead to an infinite state
space!

### Running the Model Checker

As a last deed in this section, we give the model checker something to actually
check (as of now, it would construct all behaviors, but not check anything
about them). Since we're not ready to check anything meaningful, we just assert
`TypeOK`; we'll add something else later.

```
DesiredProperties == TypeOK
```

There's no real point, but now you can try out the model. If you clone [the
repo][caspaxos-tla] and check out the [right commit][caspaxos-stub], you'll
find a prepared model along with the TLA file we've assembled so far.

Open the model using the [TLA toolbox][toolbox] (`File -> Open Spec -> Add New
Spec`), and note how we declared `Acceptors` as a set of three "model values";
a model value is a value that's completely opaque to the model checker, and not
equal to any other value. We also have `Values = {0, 1, 2, 3}`, and `Mutator(b,
v)` is set to `(1 + b*v) % 4` (an arbitrary choice).

Furthermore, there is a "Definition Override" that sets `Nat` to `0..3` and we
have added `DesiredProperties` under `Invariants` so that it will be checked
for every state reachable in this model.

Now, click "Run"... oops, immediately errors out: "Deadlock reached". This
happens because we stubbed out the actions and so from the initial state, there
is no way to satisfy the formula. That happening usually amounts to an error,
and the model checker checks for it by default. Time to add the actual
algorithm!

### Completing the Specification

We return to the currently stubbed-out actions `PrepareReq`, `PrepareRsp`,
`AcceptReq`, and `AcceptRsp`, replacing them one by one. The result will be
[this commit][caspaxos-actions].

First up is `PrepareReq`, which is the message sent by a proposer when it
gets started on a new proposal (ballot).

```
BallotActive(b) == \E m \in msgs :
                        /\ m.type = "prepare-req"
                        /\ m.bal = b
PrepareReq(b) ==
    /\ ~ BallotActive(b)
    /\ Send([
                type |-> "prepare-req",
                bal  |-> b
           ])
    /\ UNCHANGED(<<prepared, accepted, value>>)
```

A ballot is started by sending a prepare request (with the hope that
responses will be received from a quorum). We could allow multiple
prepare requests for a single ballot, but since they are all identical
and we already model multiple-receipt for all messages, this adds only
state space complexity. So a ballot will only be prepared once in this
model.

More precisely, a successor (primed) state satisfies `PrepareReq(b)` if and only if

- it's predecessor (i.e. the unprimed variables) contain no prepare request for
  ballot `b` (`~` is negation), and
- the primed state contains a message of type `prepare-req` and ballot `b`, and
- everything else is identical.

The last condition may surprise you, but consider that if we omitted it, you
could create lots of nonsensical successor states by freely mutating
`prepared`, `accepted`, and `value`. Not what we would want!

Now that we're warmed up, let's get a bit more involved and hash out
`PrepareRsp(a)`:

```
PrepareRsp(a) ==
    /\ \E m \in msgs :
        /\ m.type = "prepare-req"
        /\ m.bal > prepared[a]
        /\ prepared' = [prepared EXCEPT ![a] = m.bal]
        /\ Send([
                    acc      |-> a,
                    type     |-> "prepare-rsp",
                    promised |-> m.bal,
                    accepted |-> accepted[a],
                    val      |-> value[a]
               ])
    /\ UNCHANGED <<accepted, value>>
```

Read: A prepare response can be sent if by an acceptor if a) a response was
demanded via a prepare request (there exists a corresponding prepare request
message) and b) the acceptor has not already prepared that or any larger
ballot.  On success, the acceptor remembers that it has prepared the new
ballot, and sends a response.

The odd syntax `[prepared EXCEPT ![a] = m.bal]` means

> same as prepared, with the exception that when evaluated at `a`, return
> `m.bal`

and nobody likes it, including its author [Leslie Lamport][tla].

Phew! Getting the hang of it yet? I hope so, because now it's time for the big
leagues:

```
AcceptReq(b, v) ==
    /\ ~ \E m \in msgs : m.type = "accept-req" /\ m.bal = b
    /\ \E Q \in Quorums :
        LET M == {m \in msgs : /\ m.type = "prepare-rsp"
                               /\ m.promised = b
                               /\ m.acc \in Q}
        IN /\ \A a \in Q : \E m \in M : m.acc = a
           /\ \E m \in M :
                /\ m.val = v
                /\ \A mm \in M : mm.accepted \leq m.accepted
    /\ LET newVal == Mutator(b, v) \* crucial difference from Paxos
       IN Send([
                type   |-> "accept-req",
                bal    |-> b,
                newVal |-> newVal
               ])
    /\ UNCHANGED(<<accepted, value, prepared>>)
```

Read: An accept request can only be sent (i.e. fabricated from thin air)

1. once;
1. if prepare responses for the ballot have been received from a quorum, and
1. with a new value based on the most recently accepted value from the prepare
   responses.

It's only here that we really deviate from Single-Decree Paxos by using
`Mutator`!


Scared of `AcceptRsp`? Don't worry, that one is straightforward again.

```
AcceptRsp(a) ==
    /\ \E m \in msgs :
        /\ m.type = "accept-req"
        /\ m.bal \geq prepared[a]
        /\ prepared' = [prepared EXCEPT ![a] = m.bal]
        /\ accepted' = [accepted EXCEPT ![a] = m.bal]
        /\ value'    = [value    EXCEPT ![a] = m.newVal]
        /\ Send([
                    acc      |-> a,
                    type     |-> "accept-rsp",
                    accepted |-> m.bal
                ])
```

Read: An acceptor can reply to an accept request only if it hasn't yet prepared
a higher ballot. Before replying, it makes sure it marks the ballot as prepared
(as the particular acceptor may not have received the associated prepare
request earlier), and updates its accepted ballot and the new value.

That's it already! Try running the model again - it should take some time and
your laptop may begin radiating energy in the form of heat. On mine the state
space exploration takes around 15s and reports a total of 791370 states.

Of course, we're still only running the model, not really checking it. In the
next section, we'll add something that fails on some interesting histories.

## Discovering Interesting Histories

Now we have our system specified, a model and the associated sizable state
space to explore - but we don't actually know what we're looking for. Hence,
let's return to the algorithm at hand. The result of this section will be [this
commit][caspaxos-check].

Without formalizing it too much, we should care in one way or another whether
the "mutations line up". That is, the values that the register stores over its
(in the model, very short) lifetime should be those that `Mutator` creates
from their predecessors. Or, in the language of compare-and-swap, we shouldn't
have one value stored in the register but then be able to run
a compare-and-swap that doesn't respect this existing value.

Let's try to put this in a naive formula we can then have the model checker
verify for all states. Remember that "committed" below means that the value at
the corresponding ballot has been accepted by a majority of acceptors.

> For any committed value v' at ballot b' > 0 and the previously committed value
> v at ballot b < b', it holds that v' = Mutator(b', v).

Just to be clear, that assumption is not correct and perhaps you want to take
a minute with pen and paper and figure out why not. Or, in the spirit of this
section, let the model checker tell you why it's wrong. But first we have to
turn those words into math!

Taking the top-bottom approach, we add to our existing formula
`DesiredProperties` two to-be-defined assertions:

```
DesiredProperties == /\ TypeOK
                     /\ OnlyOneValuePerBallot
                     /\ MutationsLineUp
```

`OnlyOneValuePerBallot` is easy to define; we just want to check that we never
accept conflicting values for a proposal. It's fairly clear that that can't
happen when you look at the algorithm, but it's still good to have it checked:

```
OnlyOneValuePerBallot == \A b \in Ballot :
                            Cardinality(ValuesAt(b)) \leq 1
```

where `ValuesAt(b)` are all the values for which we've seen accept responses
for ballot `b` (i.e. the value was accepted by the acceptor sending the
message):

```
ValuesAt(b) == IF b = 0 THEN {InitialValue}
               ELSE { v \in Values :
                        \E m \in msgs :
                            /\ m.type     = "accept-rsp"
                            /\ m.accepted = b
                            /\ \E mm \in msgs :
                                /\ mm.type   = "accept-req"
                                /\ mm.bal    = b
                                /\ mm.newVal = v
```

Note how we had to comb through the accept requests to find the value; it's not
sent out with the accept response since the recipient already knows it.

Now for the prize:

```
MutationsLineUp ==
    \A b \in CommittedBallots \ {0} :
        LET newVal == UnwrapSingleton(ValuesAt(b))
            prevCommitBallot == BallotCommittedBefore(b)
            oldVal == UnwrapSingleton(ValuesAt(prevCommitBallot))
        IN  newVal = Mutator(b, oldVal)
```

where `BallotCommittedBefore(b)` is the largest ballot less than `b` for which
a value was committed (= accepted by a quorum, not necessarily `b-1`!), and
`UnwrapSingleton` turns a single-element set into its only element:

```
UnwrapSingleton(s) == CHOOSE v \in s : TRUE
AcceptedByQuorum(b) == \E Q \in Quorums: AcceptedBy(b) \cap Q = Q
CommittedBallots == {b \in Ballot : AcceptedByQuorum(b)} \cup {0}
BallotCommittedBefore(b) == CHOOSE c \in CommittedBallots :
                                /\ c < b
                                /\ \A cc \in CommittedBallots :
                                    cc \geq b \/ cc \leq c
```

### Running the Model

As before, give the model a spin - note that since we extended
`DesiredProperties`, the model checker is now checking more than before.

And, lo and behold, after working for a short while, execution stops and we see
a complaint:

> Invariant DesiredProperties is violated.

In such a situation you can get lucky and that message will be accompanied by
a "stack trace" that allows you to find out which part of the formula made it
fail. Alas, not this one, but we get the complete history of state transitions
and a trace exploration tool. From it we can transcribe (your actual history
may vary):

- ballot one and two: prepare request sent
- ballot one and two, acceptor 1: prepare response sent (existing value 0)
- ballot one, acceptor 2: prepare response sent (existing value 0)
- ballot one: accept request sent for new value `1 = Mutator(one, 0)`
- ballot one: acceptor 2: accept value 1
- ballot two: acceptor 2: **prepare response sent (existing value 1)**
- ballot two: accept request sent for new value `3 = Mutator(two, 1)`
- ballot two: acceptors 1 and 2: accept value 3
- `DesiredProperties` is violated because `3` is committed, but not equal to
  `0 = Mutator(zero, 0)`.

We'll discuss this and more in the next section.

## Discussion of CAS-Paxos

In the last section, we've discovered a history which violated our (naive) assumption that each committed value is born out of a mutation of a previous committed value: We run two ballots concurrently, and it just so happens that
without the first ballot ever committing, the second one gets to read its
value, base its mutation on it, and commit (by having two out of three
acceptors accept the ballot).

Does this mean we've found an anomaly, or, finally getting more precise, does
this mean we've found a non-linearizable history? Let's write this history up
from the perspective of linearizability:

- `time=10`: the register is initially `foo` (which means that if you run an
  instance of Single Decree Paxos, you learn the value `foo`)
- `time=10`: I dispatch `CAS(bar, baz)`
- `time=11`: You dispatch `CAS(foo, bar)`
- `time=20`: I hear back that my `CAS` went through OK.
- You never hear back.

Note that it doesn't matter which of us dispatches their `CAS` first. What
matters is that though you never learn whether your `CAS` applied, it **could**
have applied at any point in time with `time >= 11`! So it's easy to invent
legal points in time at which it applied so that things make sense:

- `time=10`: I dispatch `CAS(bar, baz)`
- `time=11`: You dispatch `CAS(foo, bar)`
- **`time=12`: Your `CAS(foo, bar)` commits**
- **`time=13`: My `CAS(bar, baz)` commits**
- `time=20`: I hear back that my `CAS` went through OK.

So this history is linearizable. Note however that that doesn't prove that the
system is! All we've managed to show is that it's not broken for this
particular example.

Looking back under the hood of how CAS-Paxos actually works, something
noteworthy *did* happen that doesn't reflect through the lens of
linearizablity: When "my" `PREPARE` phase sends its first message, you had already gone ahead and got your new value `bar` accepted on one of the acceptors:

```
Values = [bar @ 1, foo @ 0, foo @ 0]     # value @ ballot
```

When you look at the system in that state from the outside, you might be
tempted to say that the register stores the value `foo` (after all, that's the
value you can find on a majority). But that's not correct! If you want to read
the value, you must run a full phase of Single Decree Paxos. And that would
either:

- talk to a majority that has `bar`, and have `bar` committed at the end:
  
  `Values = [bar @ 2, bar @ 2, foo @ 0 or bar @ 2]`
- talk to a majority that does not have `bar`, and commit a "newer" copy of
  `foo` at the end:
  
  `Values = [bar @ 1 or foo @ 2, foo @ 2, foo @ 2]`

That is, until you actually go and read the register, it doesn't necessarily "have"
a value - if in the examples below we didn't run `ACCEPT`, you could contact
one majority first, and then the other, and you would get different values
though nobody else is using the system - definitely not linearizable!

Reading the value thus requires participating in deciding it. But it should be
possible to skip the `ACCEPT` phase if you receive the same ballot from all
possible acceptors (three in the above example).

Something similar, though less transparently, happened when my `PREPARE` phase
picked up your value which you had managed to get accepted at only one of the
nodes: It "implicitly" committed it when it committed its own update. And,
actually, since your partial value might have in turn been based on a previous
not-explicitly-committed value as well, that was also implicitly committed, and
so on. In a different possible outcome, I wouldn't have found your value, and
would have implicitly made sure that it would never be visible, as if you had
never tried to write it. That, too, is legal.

This also gives an intuitive understanding of why CAS-Paxos should be
linearizable: the state of the system after a successful CAS based on an
uncommitted value is equivalent to one in which you "make up" accept messages
to accept that value (at its home ballot) between the main operation's
`PREPARE` and `ACCEPT` phases. Then you only have to consider the case where
every value you see is committed, and that argument should be very similar to
that that assures that a Single Decree Paxos register never "loses"
a once-committed value.

```
Values = [bar @ 1, foo @ 0, foo @ 0]
Values = [bar @ 1, bar @ 1, bar @ 1] # implicit
Values = [baz @ 2, baz @ 2, foo @ 0]
```

### Runaway Commit

The fact that one proposal might pick up a previous incomplete proposal has
practical implications on the proposer running the incomplete proposal. From
its point of view, when it continues the `ACCEPT` phase, it will be told by
other acceptors that a conflicting proposal has superseded its own. But, it has
no way of finding out whether its value did make it into the system or whether
it was wiped out by the concurrent proposal. Paxos (CAS or Single Decree)
doesn't fare too well when there is a lot of concurrent proposing going on
anyway, so this may not be much of an issue (and if it were, some extra state
could be kept around by the acceptors to mitigate).

### Properly Failing a CAS

So far, we have neglected the case in which a compare-and-swap operation fails,
and in particular that in which it fails because the "previous value" is not
the required one. Naively, you would perhaps consider the following behavior:

- run a `PREPARE` phase and collect responses from a majority
- if the value with the highest ballot does not match what we expect, tell the
  client the `CAS` failed: "I tried, but the register does not hold the value
  you thought it did".

But that's incorrect! Consider again the case of a partially committed value:

```
Values = [`bar` @ 1, `foo` @ 0, `foo` @ 0]
```

A `PREPARE` phase for `CAS(foo, boo)` might pick up `bar` as the most recent
value, the client is told that the register doesn't hold the value `foo`.
But next, the client reads the register, winds up talking to the majority that
doesn't have `bar`, and learns that the value is indeed `foo` - we've got
ourselves into a nonlinearizable history.

Instead, a CAS which finds an incompatible value after `PREPARE` must run an
`ACCEPT` phase with that incompatible value, because only when that succeeds is
it allowed to tell the client with authority that the register did not hold the
correct base value. Or, it could return a more generic error to the client
which doesn't make any statement about the value of the register, but usually
you want a failed CAS to report the actual value of the register, and that
definitely needs the `ACCEPT` phase.

## Summary

We've introduced the CAS-Paxos algorithm, specified it in TLA+ and discussed
some interesting histories, phenomena, and implications on practical
implementations. Intuitively, we have seen that there is reason to believe that
CAS-Paxos provides a distributed linearizable register with compare-and-swap
semantics, which is a powerful primitive for building distributed systems.



[aphyr-lin]: https://aphyr.com/posts/313-strong-consistency-models
[crdb]: https://cockroachlabs.com
[paxossimple]: https://www.idi.ntnu.no/emner/tdt02/PaxosMadeSimple.pdf
[paxossimplearticle]: https://www.microsoft.com/en-us/research/publication/paxos-made-simple/
[tla]: http://lamport.azurewebsites.net/tla/tla.html
[toolbox]: http://lamport.azurewebsites.net/tla/toolbox.html
[rystsov-consensus]: http://rystsov.info/2017/02/15/simple-consensus.html
[rystsov-tw]: https://twitter.com/rystsov
[caspaxos-tla]: https://github.com/tschottdorf/caspaxos-tla
[mp]: https://en.wikipedia.org/wiki/Paxos_(computer_science)#Multi-Paxos
[caspaxos-stub]: https://github.com/tschottdorf/caspaxos-tla/commit/5bcac1e38a9521cc4a939103611786fd675d6ac7
[caspaxos-actions]: https://github.com/tschottdorf/caspaxos-tla/commit/f5d70bd6540bdc1d5fb5cb553ab1a4d36186fc18
[caspaxos-check]: https://github.com/tschottdorf/caspaxos-tla/commit/50229e7b8914d4d1f297940a4494b010377fccf8
