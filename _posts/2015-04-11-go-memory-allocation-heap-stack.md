---
layout: post
title: "Stack vs Heap: Improving memory allocation through benchmarks"
comments: true
permalink: "go-memory-allocation-stack-heap"
---

Recently [petermattis](https://github.com/petermattis) from [cockroach labs](http://cockroachlabs.com/) [shared](https://github.com/cockroachdb/cockroach/pull/615#discussion_r28082815) a simple method for finding weak spots in your [Go](http://golang.org) project's memory management:

> I found this using the memory profiler: `go test -bench=. -memprofile=prof -memprofilerate=1`. The `-memprofilerate=1` setting tells the memory profiler to record every allocation. This slows the benchmark down a lot, but gives a price view of where the allocations are occurring.
> I then use `go tool pprof --alloc_objects <binary> prof` to view the output.

If you've found such a spot in which the escape analysis doesn't want to put things on the stack (for example, because it's a rather large byte slice), you could use a `sync.Pool` to avoid an allocation each time the problematic code gets called, or maybe you have your concurrent access under control - then a scratch buffer is enough.
