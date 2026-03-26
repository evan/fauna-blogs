---
layout: post
title: "Fauna's Official Jepsen Results"
date: 2019-03-05
description: "I am pleased to present, along with Kyle Kingsbury of [Jepsen.io](http://jepsen.io/), the official Jepsen results for Fauna version 2.5.4 and 2.6.0. Our team at Fauna worked extensively with Kyle for three months on one o"
original_url: "https://fauna.com/blog/faunadbs-official-jepsen-results"
author: "Evan Weaver"
category: blog
---

I am pleased to present, along with Kyle Kingsbury of Jepsen.io, the [official Jepsen results](/assets/pdfs/FaunaDB_2.5.4_Jepsen.pdf) for Fauna version 2.5.4 and 2.6.0.

Our team at Fauna worked extensively with Kyle for three months on one of the most thorough Jepsen tests of all time. Our mandate for him was not merely to test the basic properties of the system, but rather to poke into the dark corners and exhaustively validate that Fauna is architecturally sound, correctly implemented, and ready for enterprise workloads in the cloud.

We’re excited to report that Fauna passed the core tests right away:

> Fauna’s core operations on single instances in 2.5.5 appeared solid: in our tests, we were able to reliably create, read, update, and delete records transactionally at snapshot, serializable, and strict serializable isolation. Acknowledged instance updates were never lost.

It now passes additional tests, covering features like indexes and temporality:

> By 2.6.0-rc10, Fauna had addressed almost all issues we identified; some minor work around availability and schema changes is still in progress.

Additionally, it offers the highest possible level of correctness:

> We expect to observe snapshot isolation at a minimum, and where desired, we can promote SI or serializable transactions to strict serializability: the gold standard for concurrent systems.

It is self-operating:

> Many consensus systems rely on fixed node membership, which is cumbersome for operators. Fauna is designed to support online addition and removal of nodes with appropriate backpressure.

And its architecture is sound:

> Fauna is based on peer-reviewed research into transactional systems, combining [Calvin](http://cs-www.cs.yale.edu/homes/dna/papers/calvin-sigmod12.pdf)’s cross-shard transactional protocol with Raft’s consensus system for individual shards. We believe Fauna’s approach is fundamentally sound...Calvin-based systems like Fauna could play an important future role in the distributed database landscape.

In consultation with Kyle, we’ve fixed many known issues and newly discovered bugs, made API improvements, and expanded our documentation. Kyle has extended the Jepsen suite itself with new tests specifically inspired by Fauna. We have also incorporated the extended Jepsen test suite into our internal QA, to help ensure that we never backtrack on the level of reliability we intend to provide.

## What Is Jepsen?

Kyle describes Jepsen as “an effort to improve the safety of distributed databases.” It is an open source software verification suite born out of industry frustration with the unsubstantiated claims made by database vendors at the dawn of the cloud era. Jepsen is now widely regarded as the critical test that any distributed system must pass before it is considered mature.

![Fauna 2.5.4 Jepsen Analysis](/assets/images/faunas-official-jepsen-results-1.png)

Those familiar with Jepsen reports will note that no other database tested has met the stringent reliability levels that Fauna has now met. The Fauna report also contains a lovely, extensive description of Fauna’s architecture, and I encourage you to read it in its entirety.

## Why Test Fauna?

When we started building Fauna, our objective was to deliver a cloud-native database that offered both transactional consistency and global scalability. For that reason, we chose Calvin as the basis for underlying transaction protocol.

Other distributed, transactional databases use the first-generation [Google Percolator model](https://blog.octo.com/en/my-reading-of-percolator-architecture-a-google-search-engine-component/), which cannot scale transactions across datacenters, or the second-generation [Google Spanner model](https://static.googleusercontent.com/media/research.google.com/en//archive/spanner-osdi2012.pdf), which [requires atomic hardware clocks](http://dbmsmusings.blogspot.com/2018/09/newsql-database-systems-are-failing-to.html) and a specialized operational environment. Fauna is the only production database to use the third-generation Calvin protocol.

By designing for global correctness up front, Fauna offers mainframe-like capabilities even in the chaos of a multi-cloud deployment. Externally consistent, multi-partition distributed transactions were widely believed to be impossible in a software-only solution until Fauna showed the way. We are proud to see our architecture validated in Kyle’s analysis.

## Summary of Correctness Tests

Jepsen’s correctness tests exercised Fauna under a wide variety of fault conditions and administrative actions to simulate the unreliable operating conditions of the public cloud, including:

* Individual process crashes
* Individual process restarts
* Rapid multi-process crashes
* Rapid multi-process restarts
* Small and large forward jumps in clock skew
* Small and large backwards jumps in clock skew
* Rapidly strobing clocks
* While undergoing log topological change
* While undergoing replica topological change

The testing validated that Fauna meets its expected isolation levels, avoids anomalies present in other databases, and maintains ACID semantics at all times. Additionally, the process of updating and running the Jepsen suite itself provided extensive verification of the general liveness, availability, and durability properties of Fauna, and let to numerous improvements.

## Ongoing Work

Fauna does not depend on clock synchronization or a central clock oracle to maintain correctness, as the Jepsen analysis shows. Databases that rely on synchronized clocks can enter a state of ambiguous, irrecoverable data corruption if clocks skew beyond tolerance. Fauna never corrupts data, regardless of skew.

Fauna versions 2.6 and earlier do partially rely on clocks to maintain liveness—the ability to process new transactions. Jepsen testing uncovered an issue where clock skews many seconds long, chaotically introduced across multiple nodes, can create cluster pauses until the skews are resolved. This operational scenario is rare in practice.

However, as Kyle notes, Fauna’s architecture makes it possible to maintain complete availability and liveness even with extreme clock skew. This is an implementation detail of Fauna rather than an architectural limitation. We look forward to proving it in an upcoming release.

## Conclusion

Since no amount of bug fixing can save the wrong architecture, we are gratified that the Jepsen report is highly complimentary of that of Fauna, and that the report validates that the issues found during testing were rapidly fixed:

> The bugs that we’ve found appear to be implementation problems, and Fauna has shown a commitment to fixing these bugs as quickly as possible.

We look forward to working with Kyle and the Jepsen team in the future as we make further improvements to Fauna’s architecture and implementation. In the meantime, go read the full report!