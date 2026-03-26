---
layout: post
title: "A comparison of transaction models in distributed document databases"
date: 2021-07-28
description: "The truly serverless database that combines the power of a relational database with the flexibility of JSON documents."
original_url: "https://fauna.com/blog/comparison-of-transaction-models-in-document-databases"
author: "Evan Weaver"
category: blog
---

*This is the first in a series of posts comparing different aspects of modern operational databases.*

A distributed document database is a NoSQL database that stores semi-structured data, usually in a format similar to JSON, and is horizontally scalable across multiple machines. Document databases are very convenient for modern application development, because modern application frameworks and languages are also based on semi-structured objects instead of tabular data. They are often a better fit than a SQL database, where rigid schemas resist object evolution, and complex client-side object-relational mappers simulate an object-oriented interface with incomplete success.

In the distant past of the 2010’s, distributed document databases didn’t offer [transaction support](https://blog.couchbase.com/couchbase-transactions-with-n1ql/), instead implementing various forms of eventual consistency. Vendors and open-source maintainers promoted the idea that transactionality was an unnecessary, complexifying feature that damaged scalability and availability—and many claimed that adding it to their systems was [impossible](https://stackoverflow.com/questions/16779348/does-the-cap-theorem-imply-that-acid-is-not-possible-for-distributed-databases).

Time has gone by and things are very different now. The benefits of transactionality are widely accepted outside of the context of SQL and [even databases in general](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/saga/saga), and key distributed systems problems have been solved. All modern distributed document databases now offer some form of transactionality, but their [implementation](https://static.googleusercontent.com/media/research.google.com/en//archive/spanner-osdi2012.pdf)s and characteristics vary widely.

Read on to explore the differences between Couchbase, MongoDB, Google Cloud Firestore, and Fauna.

### What is best in life and transactions?

Database transactions keep changes to disparate records bundled together, ensuring that applications can read and write data without colliding with each other or interleaving their updates in confusing or invalid ways.

The ACID acronym (“atomicity, consistency, isolation, durability”) is commonly used to describe the basic properties of transactions, but doesn’t fully describe what it’s like to use them. In practice, when evaluating transaction implementations, we want to look for:

* **Consistency level:** higher consistency levels are easier to reason about
* **Generality:** fewer special cases in implementation and interface lead to fewer bugs
* **Performance:** the less that transactions impact throughput and latency, the better
* **Ease-of-use:** writing transactional queries should be straightforward, and reading transactional data should be transparent

We will keep this framework in mind as we discuss the various systems.

## Couchbase

Couchbase was originally developed by merging Membase, a fork of Memcache with on-disk durability, with CouchDB, a somewhat experimental document database written in Erlang. As such, it has a relatively tortured development history. It has adopted a relatively tortured transaction model as well.

Couchbase transactions work [purely from the client-side](https://blog.couchbase.com/distributed-multi-document-acid-transactions/). Each Couchbase SDK implements an algorithm that essentially writes a lock record into each shard, then writes uncommitted writes into each document as metadata, then updates the lock records, then moves the metadata into the documents to become the primary data. Other clients that observe document metadata must check the locking document to find out the state of the transaction, and race to clean up unapplied or aborted writes based on wall clock time.

### ACID properties

Couchbase’s algorithm provides read-committed isolation as far as document reads go, but no isolation at all for [derived data like indexes](https://docs.couchbase.com/server/current/learn/data/transactions.html#indexes-and-transactions), which have no way to tell if data is being modified concurrently or not. It also does not offer snapshot isolation for readers, which means that nobody really has a consistent view of the entire database at any time, and it is vulnerable to clock skew in the locking mechanism.

As far as I can tell, when Couchbase transactions span shards, they are not actually isolated or atomic, because there is no specific reason to assume that readers will observe transaction record state in the same order that writers do, or that a writer will successfully update all locking documents.

Couchbase offers several tunable durability and quorum options that interact with transaction read and write consistency in confusing ways, and the default does not require disk persistence for writes. Finally, multi-region transactions are not supported at all because there is no synchronous multi-region replication.

Couchbase has not verified their transactions under Jepsen, but only done some [in-house work](https://blog.couchbase.com/introduction-to-jepsen-testing-at-couchbase/) to verify the basic single-document replication protocol. Ultimately, Couchbase does not offer any meaningful general level of consistency, so maybe there’s nothing to verify. For example, you cannot use this system to implement unique constraints, a very basic goal.

### Interface

Couchbase has adopted a SQL-like syntax called N1QL and applied the same tactic to transaction support, with explicit START TRANSACTION/COMMIT TRANSACTION statements. Transactions are interactive in that clients can hold them open indefinitely (well, as long as the cleanup window lasts) and interleave them with client-side computation.

Here is a N1QL example:

```
START TRANSACTION;
SELECT COUNT(*) FROM airport WHERE city='Stanted';
UPDATE airport SET city='London' WHERE faa='STN';
DELETE FROM airport WHERE city='London' AND faa != 'STN';
COMMIT TRANSACTION;
```

This interface is reasonable, although many other Couchbase interfaces, and even other SDKs, are not transaction-aware and will serve stale reads during commits.

### Performance

Because the transactional operations involve so many additional client-side reads and writes, they are much slower than normal operation, which Couchbase touts as a feature—because you can avoid the slowdown by not using transactions.

### Summary

So how does Couchbase fare according to our criteria? The consistency level offered is essentially no consistency at all by database standards. Generality is poor since transactions due not affect indexes or non-transactional clients. Theoretical performance is also poor relative to baseline due to the multiphase client-side lock management. Ease-of-use is adequate, though, as long as the interface supports it.

Personally, I feel like Couchbase transactions should really be called batch writes, as they offer essentially none of the properties one would expect from a multi-document transaction.

## MongoDB

MongoDB began its life essentially as a JSON database for node.js. Very easy to use for JavaScript developers, it struggled with basic durability until the addition of the WiredTiger storage engine. After many years of preaching that transactions were unnecessary and advocating for extensive data denormalization to compensate, MongoDB capitulated and added transaction support a few years ago.

MongoDB implements a sharded primary-secondary replication system, with a dizzying array of [tunable read and write consistency levels](https://developer.mongodb.com/how-to/global-read-write-concerns/#overriding-global-read-and-write-concerns). Its transactions are implemented via a cluster-wide [hybrid logical clock](https://www.mongodb.com/blog/post/transactions-background-part-4-the-global-logical-clock) from which transaction timestamps are derived. Within a shard, transactions are written first to a write-ahead log on the primary node, and then the values of the transactions are written with their timestamps into the storage engine, which offers multi-version concurrency control for snapshot reads. Transactions that span shards have an additional layer of what appears to be two-phase commit coordination.

### ACID properties

At best, MongoDB offers snapshot isolation. This means that unlike Couchbase, it at least has some hope of being serializable (which basically means transactions get applied in some non-overlapping order, even if it wasn’t the one you would expect).

Unfortunately, the default consistency levels allow reads of committed data to happen before replication has occurred, so dirty reads and lost transactions are possible during failover. Like Couchbase, even this level of fault-intolerant isolation is limited to a single shard. For transactions which span shards, [there is no read-side coordination](https://docs.mongodb.com/manual/core/transactions-sharded-clusters/): readers will observe transactions torn at their shard boundaries, violating snapshot consistency. Even readers within transactions will not see isolated commits of other transactions unless a non-default read consistency level is used.

Unique constraints are supported, but apparently via a different mechanism than transactions as they predate them. It is unclear to me how indexes with unique constraints interact with snapshot isolation within transactions.

MongoDB has been independently tested with Jepsen, but there is [extensive disagreement](https://jepsen.io/analyses/mongodb-4.2.6) between the company and the Jepsen team about the correct interpretation of the results. There are [many combinations](https://docs.mongodb.com/manual/core/transactions/%23transactions-and-write-concern) of read and write consistency levels that will violate even the most basic transactional properties.

### Interface

Rather than a true query language, MongoDB implements DSLs in its drivers that follow the same basic syntactic patterns. Beginning and ending a transaction requires creating a session object with correct read and write consistency levels, and then separately beginning a transaction with (possibly different) read and write consistency levels, and then committing the transaction. Naively using the transactional feature will not reliably result in transactional behavior.

Here is a [JavaScript example](https://docs.mongodb.com/v4.0/core/transactions/):

```
session = db.getMongo().startSession( { readPreference: { mode: "primary" } } ); session.startTransaction( { readConcern: { level: "snapshot" }, writeConcern: { w: "majority" } } );
session.getDatabase("hr").employees.updateOne( { employee: 3 }, { $set: { status: "Inactive" } } );
session.getDatabase("reporting").events.insertOne( { employee: 3, status: { new: "Inactive", old: "Active" } } );
session.commitTransaction();
```

Additionally, out of band coordination, most likely via human beings, is required to make sure that all potential readers of the transactional writes are also transaction-aware.

### Performance

MongoDB transactions add little overhead to single shard queries in their default configuration, but also don’t really offer any transactional properties. Setting the appropriate read and write consistency levels means that both readers and writers will block on quorum among the primaries and secondaries of the shard. This impacts the latency profile of writes, and the throughput and latency profile of reads, since more nodes have to be queried.

Moving to sharded queries slows down writes substantially due to the extra roundtrips the two-phase model layers on top. Unless transactions and shard keys are carefully designed to avoid this, the performance discontinuity may be unpredictable.

### Summary

MongoDB’s consistency level is adequate, but the generality is poor because of all the edge cases. Theoretical performance is poor and unpredictable relative to baseline. And due to the confusing interaction of the read, write, and transaction consistency, ease-of-use is also not good, putting aside any issues with the query language itself.

However, I will grant that unlike Couchbase, MongoDB transactions really are transactions.

## Google Cloud Firestore

Some databases are actually domain-specific interfaces to other databases, and Firestore is one of them. Firestore is the updated version of [Firebase Realtime Database](https://firebase.google.com/docs/database), a mobile-oriented cloud service that was originally implemented as a service on top of MongoDB. Google acquired Firebase in 2014, and ported the interface, with modifications, to a [Google Spanner](https://cloud.google.com/spanner) backend in 2019.

Firebase implemented an unusual hierarchical data model, somewhat similar to [MUMPS](https://en.wikipedia.org/wiki/MUMPS). This caused a lot of problems because it was difficult to query and update subtrees of the hierarchy in a predictable and performant way. Nevertheless, treating the entire database as a single document did allow for atomic updates of subtrees.

Firestore improved on the Firebase data model by switching to a [hybrid document-hierarchical model](https://firebase.google.com/docs/firestore/data-model). Instead of a giant tree, the database is now composed of collections, which can contain documents, which can contain subcollections, which can contain subdocuments, more or less indefinitely. This creates natural record boundaries at the document level which corresponds well with the underlying relational model of Spanner.

This data model was influenced by the Google Cloud Datastore data model, an eventually consistent document database originally built for Google App Engine that also predated Google Spanner. In fact, Firestore includes a Datastore compatibility mode.

### ACID properties

Conceptually, Firestore is rewriting document queries into relational queries for Spanner, and inherits Spanner’s consistency model. Spanner implements serializability via a multi-phase implementation backed by physical atomic clocks. Serializability is a very good isolation level.

However, there are some nuances here. Due to its origin as a mobile database, Firestore is designed around mixed client access. Because mobile devices are usually highly latent, Firestore transactions are implemented in mobile clients with client-side [optimistic concurrency control](https://firebase.google.com/docs/firestore/transaction-data-contention#concurrency-controls). Effectively, all mobile transactions are rewritten into a series of incremental reads, and then a transactional compare-and-swap operation for the writes. This has a number of implications:

* In a Firestore transaction, [all reads are executed before all writes](https://firebase.google.com/docs/firestore/manage-data/transactions#transaction_failure)
* Reads cannot see the result of writes within the transaction
* A transaction, including its client side logic, may run multiple times, so side effects are dangerous

On the other hand, server-side transactions use a more typical pessimistic relational lock. They open a transaction, do some work, and then commit it.

Read-only queries appear to default to strong consistency, with no way to drop to snapshot consistency like Spanner, unless you use the alternate Datastore API.

Firestore mobile clients also use a local cache for offline syncing. This means that it is possible for several things to happen when the client is offline:

* Stale reads will be served from cache
* Non-transactional writes and write-only transactions will be buffered and submitted when the client reconnects
* Read-write transactions will error out

Although these are desirable behaviors for Firestore, they violate the consistency model.

Firestore has not been verified with Jepsen, but there is no serious doubt in the industry that Spanner’s implementation is sound. Nevertheless the mobile clients clearly violate the database’s consistency guarantees.

### Interface

Firestore’s interface is composed around transaction lambdas, which can intermix client-side compute with simple calls for database reads and writes. As discussed above, whether the transaction executes optimistically and potentially multiple times, or pessimistically and only once, depends on whether a server or mobile driver is used.

Here is a JavaScript example:

```
db.runTransaction((transaction) => {
    return transaction.get(db.collection("cities").doc("SF")).then((sfDoc) => {
        var newPopulation = sfDoc.data().population + 1;
        transaction.update(db.collection("cities").doc("SF"), { population: newPopulation });
    });
});
```

Like MongoDB, Firestore does not offer a true query language, and supports quite a bit less in terms of query sophistication. As far as transactions go, this interface is reasonable, but the inability to read your own writes within a transaction is an inconvenience.

Firestore appears to [implement its indexes](https://firebase.google.com/docs/firestore/query-data/indexing) as separate tables within Spanner. It requires the user to explicitly lay out the index data to support the application’s query patterns, rather than delegating to indexed relational queries underneath. Firestore indexes every individual field within a document by default, to make naive querying easier. This comes at a cost: transaction sizes by default are relatively large. It is possible to manually exempt fields from indexing.

### Performance

Firestore offers many Google Cloud single-datacenter [locations](https://firebase.google.com/docs/firestore/locations), but only multi-datacenter support in the US and Europe. Multi-datacenter support comes with a higher cost and higher write latency, but potentially lower read latency depending on client location.

Firestore transactions run in Spanner’s default strong consistency mode, which is fast, especially for single-datacenter databases. Multi-datacenter transactions must block on a write quorum for every shard in the query. How indexing affects the shard layout within a transaction is unclear.

### Summary

Firestore’s consistency model, after the rewrite onto Spanner, is very good. Generality is optimal with no meaningful edge cases except for well-documented and predictable ones. However, ease of use could be better. Performance is good. But the data model and indexing scheme are strange, even if they are familiar to longtime Firebase users. And the lack of query flexibility limits utility compared to something like SQL or N1QL.

## Fauna

Fauna is a database as an API. Is it a document database? Yes, and it is also a document-relational database, since it offers not just transactions, but foreign keys, views, and joins, and also provides temporal access to data history.

Fauna implements a unique single-phase transaction architecture based on Calvin. Like MongoDB, its data model is based on documents and collections. Like Spanner, it offers a best-in-class consistency model. Like Firestore, it has a serverless operational model. And unlike other document databases, it offers the ability to compute close to the data with a rich standard library and function model—as rich or richer than SQL.

### ACID properties

In Fauna, every query is a transaction. Read-write transactions are strictly serializable via the transaction log, which offers a partitioned and replicated commit point without coordination with the data replicas or with atomic clocks. Read-only transactions are serializable via careful timestamp management within the database and the database drivers, and in practice, indistinguishable from strict serializability.

Unlike the other databases here, there are no default consistency level concerns or driver features that can violate the consistency model. Transactions can read their own writes as well.

The one thing transactions can’t do is client-side compute; instead, Fauna focuses on in-database computation, with a Turing-complete query language called FQL and support for complex branching and looping, math and string manipulation, and user-defined functions.

Fauna’s consistency model and implementation have been independently verified with Jepsen and found to be sound, although the analysis is out of date and the database is much improved since.

### Interface

Like Firestore, Fauna’s interface is composed around query lambdas. However, within the lambda, the user cannot make application-side calls with side effects or that have read or write dependencies. Instead, this work is delegated to the database itself.

Here is a [Javascript example](https://fireship.io/lessons/fauna-db-quickstart/):

```
await client.query(
        Create(
            Collection('relationships'),
            follower: Call(Fn("getUser"), 'alice'),
            followee: Call(Fn("getUser"), 'bob')
        )
    )
});
```

This query is transactional by default and refers to a user-defined database function “getUser”, part of this database's schema, to simplify the logic.

Although side-effects and computation on dependencies are not allowed, client language features can still be used for query composition; for example, defining a subquery ahead of time and referring to it with a local variable.

Although Fauna encourages an index-per-query pattern similar to Firebase, it is not required, and Fauna does not automatically create multiple default indexes for documents.

### Performance

Fauna is designed to minimize latency in multi-datacenter replication scenarios. A read-write transaction only requires one roundtrip within a majority of replication sites, and read-only transactions do not require any round trips at all. This makes the performance profile similar to Spanner and presumably to Firestore as well.

Fauna does not yet offer single-datacenter region configurations, so its latency will be higher than other databases configured without multi-region replication.

Couchbase, MongoDB, and Firestore all use an interactive session model: reads and writes are issued incrementally, and compute is run in the client. This means that latency stacks up more or less linearly with the number of database operations within the transaction.

Fauna, on the other hand, includes a comprehensive standard library within FQL itself. This allows the database to execute business logic colocated with the data, and also to guarantee that the logic runs only once and is side-effect free.

### Summary

At Fauna, we want to give developers the best database infrastructure possible delivered by the ideal APIs, especially document APIs FQL. The Fauna architecture offers the highest level of consistency, strict serializibility, without edge cases. Performance is very good and theoretically optimal in multi-region scenarios. And since Fauna is delivered as an API, our customers are responsible for no operational tasks of any kind, even though the underlying service is continuously being improved.

Although FQL is unfamiliar to most, we are working hard to improve its ease of use and expand its capability. Certainly taking advantage of its transactionality is trivial because everything is a transaction by default, with no “escape routes” that could lead to potential correctness problems.

If you are looking for a document database, I hope this article helps guide you in the right direction. And if you want to never have to manage that database again, don’t forget to give Fauna a try.