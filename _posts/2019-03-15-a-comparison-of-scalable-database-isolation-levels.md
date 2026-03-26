---
layout: post
title: "A Comparison of Scalable Database Isolation Levels"
date: 2019-03-15
description: "It is very difficult to find accurate information about the correctness and isolation levels offered by modern distributed databases and the operational conditions required to achieve them. Developers"
original_url: "https://fauna.com/blog/comparison-of-scalable-database-isolation-levels"
author: "Evan Weaver"
category: blog
---

It is very difficult to find accurate information about the correctness and isolation levels offered by modern distributed databases and the operational conditions required to achieve them. Developers use different terms for the same thing, the meaning of terms varies or is ambiguous, and sometimes vendors themselves do not actually know.

At Fauna, we care a lot about accurately describing which guarantees different systems provide. This is our effort to centralize a description of which database does what. For consistency's sake, we will use the terminology from Kyle Kingsbury's [explanation on the Jepsen site](https://jepsen.io/consistency). The chart is ranked by the maximum multi-partition isolation level offered.

The data is based on statements about isolation levels from vendor documentation, white papers, and developer commentary, exclusive of aspirational marketing statements. We have tried to be neutral in the characterization of the various systems' architectural properties. Whether the system implementations uphold these guarantees is [addressed elsewhere](https://jepsen.io/analyses). If you haven't already, please see Fauna's own Jepsen results for confirmation that Fauna upholds its guarantees.

## Before we BEGIN

In discussing transactional isolation, we frequently encounter the "worse is better" argument, which essentially goes:

* This database does what it does
* Implementing better isolation in the *database
* is impossible or has unacceptable tradeoffs
* Implementing better isolation in the *application
* is simple and useful

This argument also goes by "it's not a bug, it's a feature".

The pretense of low maximum isolation levels, eventual consistency, or [CRDTs](http://christophermeiklejohn.com/erlang/lasp/2019/03/08/monotonicity.html) is that application developers are ready and willing to work through every failure and recovery condition of their distributed dataflow. But in practice, moving beyond "works on my machine" correctness testing requires an extraordinary level of investment that product teams simply can not do.

In my experience, the implications of different isolation levels are very subtle. Pushing the burden to application developers—especially when there are a lot of distinct applications, like in a microservices architecture—is tremendously detrimental to productivity. And although tunable consistency increases flexibility, it cannot be used to paper over an isolation level that is fundamentally too weak to effectively compose.

After all, /dev/null is serializable, but not very useful as a database.

## Distributed Databases

Distributed databases present a unified topology and do not require operator management of replication, although some, like the Percolator systems, do require management of special nodes.

| System | Maximum isolation level | Default isolation level | Minimum isolation level | Consensus architecture | Limitations |
|--------|-------------------------|-------------------------|-------------------------|------------------------|-------------|
| Fauna | [Strict serializability](https://jepsen.io/consistency/models/strict-serializable) | [Snapshot](https://jepsen.io/consistency/models/snapshot-isolation) | Snapshot | Calvin | Read/write transactions without indexes are always strictly serializable. Writes must coordinate on local log leaders. Reads can be served from any replica. |
| Google Cloud Spanner | Strict serializability (and "external consistency") | Strict serializability | Snapshot (called "bounded staleness") | Spanner | Writes must coordinate on partition leaders which may be remote. Reads can be served from any replica. |
| FoundationDB | Strict serializability | Strict serializability | Snapshot | Modified Percolator | All queries must coordinate on the timestamp oracle. Conflict resolution is deferred until commit. |
| CockroachDB | [Serializable](https://jepsen.io/consistency/models/serializable) | Serializable | Serializable | Modified Spanner | All queries must coordinate on the partition leaders for their respective keys. Transactions with shared keys are mutually serializable, but transactions with disjoint keys can suffer "causal reversal". Isolation is violated under clock skew. |
| Yugabyte | Snapshot | Snapshot | Snapshot | Spanner | Isolation is violated under clock skew. |
| TiDB | [Repeatable read](https://jepsen.io/consistency/models/repeatable-read) | Repeatable read | Repeatable read | Percolator | All queries must coordinate on the timestamp oracle. |
| DynamoDB | Strong partition serializability | [Read committed](https://jepsen.io/consistency/models/read-committed) | Read committed | Paxos | Multi-partition two-phase commit offers limited serializability support. Multi-partition transactions limited to 10 primary keys with explicit read dependencies. Indexes are not serializable. Isolation is violated if there are non-transactional queries to the same keys or if global tables are used. |
| CosmosDB | Strong partition serializability | Linearizable for single-region, snapshot for multi-region | [Read uncommitted](https://jepsen.io/consistency/models/read-uncommitted) | Paxos | Multi-partition transactions are not supported. |
| Cassandra | Strong partition serializability | Read uncommitted (aka "eventual consistency") | Read uncommitted | Single-decree Paxos | Multi-partition transactions are not supported. Isolation is violated if there are non-transactional queries to the same keys, or if global secondary indexes are used. |
| MongoDB | [Session causality](https://jepsen.io/consistency/models/writes-follow-reads) | Read uncommitted | Read uncommitted | Sharded, semi-synchronous replication with automated failover | Multi-partition transactions are not supported. Isolation is violated during partitions and shard leader election. |

## Replicated Databases

Replicated databases require operator management of primaries and secondaries and the associated replication links. Asynchronous replication can improve availability and scale read capacity, but does not offer any distributed consistency guarantees. Semi-synchronous replication further improves availability, but does not improve distributed isolation.

This is the traditional RDBMS scale-out model.

| System | Maximum isolation level | Default isolation level | Minimum isolation level | Replication architecture | Limitations |
|--------|-------------------------|-------------------------|-------------------------|--------------------------|-------------|
| Oracle | Snapshot | Snapshot | Read committed | Asynchronous replication | Oracle's SERIALIZABLE isolation is not serializable, but is actually snapshot isolation with write conflict detection. This allows write skew anomalies. |
| MySQL | Serializable, primary node only | Repeatable read, primary node only | Read uncommitted | Semi-synchronous replication | |
| PostgreSQL | Serializable, primary node only | Read committed | Read committed | Semi-synchronous replication | |

## Conclusion

A good way to think about isolation is in terms of the breadth of potential anomalies. The lower the isolation level, the more types of anomalies can occur, and the harder it is to reason about application behavior both at steady-state and under faults. At Fauna, we encourage you to think critically about whether your current databases really guarantee the level of transactional isolation you need.

## References

1. [Calvin: Fast Distributed Transactionsfor Partitioned Database Systems](/assets/pdfs/calvin-sigmod12.pdf)
2. [Spanner: Google's Globally-Distributed Database](/assets/pdfs/spanner-osdi2012.pdf)
3. [Large-scale Incremental Processing Using Distributed Transactions and Notifications](/assets/pdfs/Peng.pdf)
4. [Cloud Spanner: TrueTime and external consistency](https://cloud.google.com/spanner/docs/true-time-external-consistency)
5. [FoundationDB Consistency](https://apple.github.io/foundationdb/consistency.html)
6. [FoundationDB Record Layer:A Multi-Tenant Structured Datastore](/assets/pdfs/foundationdb-record-layer.pdf)
7. [CockroachDB's Consistency Model](https://www.cockroachlabs.com/blog/consistency-model/)
8. [Jepsen CockroachDB beta-20160829](https://jepsen.io/analyses/cockroachdb-beta-20160829)
9. [YugaByte Isolation Levels](https://docs.yugabyte.com/latest/architecture/transactions/isolation-levels/)
10. [TiDB Transaction Isolation Levels](https://github.com/pingcap/docs/blob/8dd9b3e/sql/transaction-isolation.md)
11. [Consistency levels in Azure Cosmos DB](https://docs.microsoft.com/en-us/azure/cosmos-db/consistency-levels)
12. [Transactions and optimistic concurrency control](https://docs.microsoft.com/en-us/azure/cosmos-db/database-transactions-optimistic-concurrency)
13. [How are Cassandra transactions different from RDBMS transactions?](https://docs.datastax.com/en/cassandra/3.0/cassandra/dml/dmlTransactionsDiffer.html#dmlTransactionsDiffer__isolation)
14. [Jepsen MongoDB 3.6.4](https://jepsen.io/analyses/mongodb-3-6-4)