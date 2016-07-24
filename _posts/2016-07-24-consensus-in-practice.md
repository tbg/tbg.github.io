---
layout: post
title: "Distributed Consensus In Practice"
comments: true
permalink: "distributed-consensus-in-practice"
---

*This post originally appeared on the [Cockroach Labs blog](https://blog.cockroachlabs.com) as [Consensus, Made Thrive](https://www.cockroachlabs.com/blog/consensus-made-thrive/).*

When you write data to [CockroachDB](https://github.com/cockroachdb/cockroach) (for example, if you insert a row into a table through the [SQL client](https://www.cockroachlabs.com/docs/use-the-built-in-sql-client.html)), we take care of replication for you. To do this, we use a [consensus protocol](https://en.wikipedia.org/wiki/Consensus_(computer_science)) – an algorithm which makes sure that your data is safely stored on multiple machines, and that those machines agree on the current state even if some of them are temporarily disconnected.

In this post, I will give an overview of common implementation concerns and how we address these concerns in CockroachDB. Then I will abandon these earthly constraints and explore how we could improve consensus algorithms. Specifically, what would it take to make them faster?

## Consensus Algorithms Applied

Consensus algorithms are inherently distributed, and the problem they solve is fundamental to any piece of software which wants to keep a consistent state across multiple machines. After several decades, the body of research on them seemingly presents you with a variety of implementations options to choose from. However, as pointed out in [Google’s “Paxos Made Live”](http://research.google.com/archive/paxos_made_live.html), *using consensus algorithms in the real world is not quite as simple*: the things that matter most for real implementations are often mere side notes in their respective papers.

A typical consensus algorithm accepts operations from a client, and puts them in an ordered log (which in turn is kept on each of the replicas), acknowledging an operation as successful to the client once it is known that the operation has been persisted in a majority of the replicas’ logs. Each of the replicas in turn execute operations from that log in order, advancing their state. This means that at a fixed point in time, the replicas may not be identical, but they are advancing through the same log (meaning that if you give them time to all catch up, they will be in the same state) – the best you can hope for in a distributed system.

## Typical Concerns When Implementing a Consensus Algorithm:

Log Truncation: Having all of the operations in an ordered log is fine, but that log can’t grow forever! When all replicas have caught up, older log entries should be discarded.
Snapshotting: Since the log can’t be kept forever, after an extended period of downtime of a replica, there must be an alternative way of catching it up. The only option is transferring a snapshot of the data and a log position from which to resume.
Membership Changes: These are very tricky to get right. As we add a replica to the group, the size of a majority changes. A lot of decisions have to be made: which majority size is active while the membership changes? Does the new replica have any say in the group while it’s being added? When does it receive a snapshot? Can a snapshot be sent before the membership change is carried out, to minimize the impact of the change? Removal is similarly iffy, and the consensus group is typically more vulnerable while the process is ongoing.
Replay Protection: commands proposed by a client may be executed multiple times (or never, depending on the implementation). While one client proposal ideally leads to exactly one executed command in almost all cases, [general exactly-once delivery is impossible in a distributed system](http://bravenewgeek.com/you-cannot-have-exactly-once-delivery/). In practice, this means keeping state about already executed commands, or even better, using only idempotent commands.
Read Leases: when using a vanilla consensus protocol, all read operations of the replicated state have to go through that consensus protocol, or they may read stale data<sup>[1]</sup>, which is a consistency violation. In many applications, the vast majority of operations are reads, and going through consensus for those can be prohibitively expensive.
These and many others (which aren’t as readily summarized) make it hard to work an instance of a consensus protocol into a real application, let alone thousands of them.

## “Raft Made Live”

At CockroachDB, we have most of the above concerns sufficiently covered. The author of Raft, our consensus protocol of choice, did a pretty good job at providing comprehensive instructions for much of the above. We truncate our log appropriately, regardless of whether all replicas [are up or not](https://github.com/cockroachdb/cockroach/pull/7438). We send snapshots when appropriate, and soon we will also send [pre-emptive snapshots](https://github.com/cockroachdb/cockroach/pull/7468) during membership changes. We implement replay protection using [MVCC](https://en.wikipedia.org/wiki/Multiversion_concurrency_control) and a [consensus-level component](https://github.com/cockroachdb/cockroach/pull/6961). And, last but not least, we have a stable leading replica which gets to serve reads locally.

That’s all fine and well, but there are various areas of improvement. Let’s leave behind the realm of what’s been implemented (at least in CockroachDB, and probably almost everywhere else) and talk about what should be possible in an ideal world.

## Consensus is like caviar: too expensive to splurge on

The most obvious problem with distributed consensus is that it’s inherently slow. A typical consensus operation goes as follows:

```
                                  CLIENT
                                    |  ʌ
                                (1) |  | (4)
                                    v  |
                                   LEADER
                               [node1, Oregon]
                                 /  ʌ    \
                          (2)  /  / (3)    \ (5)
                             /  /            \
                           v  /                v
                        FOLLOWER            FOLLOWER
                   [node2, California]  [node3, Virginia]
                                         (responds later)
```

A client sends a request to the leader. In turn, the leader must talk to a majority of nodes (including itself), i.e. in the picture it would have to wait for one of the followers (for simplicity we assume that node three is always last). In simple math, assuming that actual message processing takes no time, we get

`commit_latency = round_trip(client, leader) + round_trip(leader, follower)`

This internal coordination is expensive, and while it’s unavoidable, we can see that the price tag depends heavily on the location of the client. For example, with a client in Oregon, we have roughly zero latency from the client to the leader, and ~30ms round-trip between the leader and the follower in Virginia, for a total commit latency of about 30ms. That doesn’t sound so bad, but let’s look at a client on the east coast instead – it would presumably be close to our Virginia data center, but that doesn’t matter – the leader is in Oregon, and we pay perhaps an 80ms round trip to it, plus the same 30ms as before, adding up to a hefty 110ms.

This goes to show that once you have consensus, you will do all you can to reduce the time you wait for those transcontinental (or even transmundial) TCP packets. For example, you could ask yourself why in that last example the client couldn’t talk directly to the node in Virginia.

There is a relatively recent consensus protocol called EPaxos which allows this<sup>[2]</sup>, though we’ll save it for another blog post. Today, we’re going to deal with a more modest question:

## Can we make reads cheaper?

Read operations may seem innocuous at first. They get served from the leader because that replica is the only one that can guarantee that it’s not reading stale data (since it decides when write operations commit), but read operations don’t have to go through consensus themselves. This means that for our example, we shave 30ms of the commit latency if we only read data. However, reads are still expensive when you’re far away from the leader. It seems silly that the client in Virginia can’t read from its local node; sure would be nice to do better, right? And you can! (At least in the literature.)

The idea is simple:

>
By letting the consensus group agree that commands must be committed by a special majority of the nodes as opposed to any majority, the nodes in that special majority can be sure to be informed about the latest state of the system.
For example, in the above picture, we could set things up so that all nodes agree that nodes one and two are to be included in any majority when committing commands (regardless of who the leader is), and then these replicas could serve reads which are consistent with the consensus log. The resulting algorithm is investigated (in higher generality) in [Paxos Quorum Leases: Fast Reads Without Sacrificing Writes](https://www.cs.cmu.edu/~dga/papers/leases-socc2014.pdf).

In a simpler world, modulo the usual implementation headaches, that could be the end of the story – but there’s an additional bit of complexity hidden here: the fact that CockroachDB is not a simple replicated log, but a full-fledged MVCC database. This means that the key-value pairs we store have a logical timestamp attached to them, and the one invariant that we must uphold is the following:

> If a value was ever read at some timestamp, it can only be mutated at higher timestamps.

That makes perfect sense if you think about it – if you read a certain value at some timestamp, then I should not be able to perform a write that changes the value you already observed.

On the leader, this is achieved by keeping an in-memory timestamp cache – a data structure which given a key and a timestamp will tell you whether the key was read at a higher timestamp previously. This structure must be consulted before proposing a write to consensus to guard us against the scenario described above – if there was such a read, we can’t perform the write.

If local reads were served at another replica, naively the leader would have to be notified about that synchronously (in order to write to the timestamp cache) before returning the result of the read to the client – the very thing we wanted to avoid! Or, somewhat better, we could let reads remain cheap for the most part and shift complexity onto writes, requiring them to contact the special majority before proposing to confirm that writing at this timestamp is still possible, and prompting the special majority to not serve reads with conflicting timestamps (at least until they see our command pop out of the consensus protocol).

Another (much more complicated) option is to incorporate that feature at the consensus level by allowing replicas to reject commands before the commit. In that scenario, roughly the following would occur:

1. Follower 1 serves a local read at timestamp, say, `ts=15`.
1. A client asks the leader to write that same key at timestamp `ts=9`.
1. The leader proposes a corresponding command to consensus.
1. The consensus algorithm on the leader tries to replicate this command to a majority of followers (including the special majority).
1. Each follower checks the command for timestamp cache violations. Some followers may acknowledge the proposal, but on Follower 1, it is rejected due to already having served a read for `ts=15` prior to the write at `ts=9`.
1. The leader, upon receiving the rejection, informs the client and cancels replication of the command suitably (either by turning it into a no-op or by unregistering it completely, depending on what’s possible).

To the best of my knowledge, such an addition has not been considered for any consensus protocol (and in particular not for the one of most interest to us, Raft). Allowing individual replicas to reject certain commands ad-hoc (i.e. basing their decision on auxiliary unreplicated state) must be considered very carefully and adds considerable complexity (in particular when leadership changes as these commands are in flight).

Performing that work is likely a small research paper and a bunch of implementation, but in contrast to many other more complicated endeavor, it seems within reach (and with it, serving local reads from some replicas).

## Conclusion

Our usage of consensus algorithms in CockroachDB is fairly standard and covers all the basic needs – but taking a step up is something we’ll be working on in the future. While the likely next step is serving reads from (some) followers, techniques which save round-trips on writes are also appealing, but those go extremely deep down the rabbit hole (and have new, much deeper challenges with respect to serving local reads). As usual, distributed consensus is hard. And if that’s just your cup of tea, [you could have that tea every day](https://www.cockroachlabs.com/careers/).

[1] Even if the client attempts to talk to the master node, the node it talks to may not be the actual master (though it may think it is), and so commands which have already committed and influenced the outcome of our read may not yet have been executed on the node we’re reading from yet – this violates linearizability on a single register.

[2] [Egalitarian Paxos (There Is More Consensus In Egalitarian
Parliaments)](https://www.cs.cmu.edu/~dga/papers/epaxos-sosp2013.pdf)
