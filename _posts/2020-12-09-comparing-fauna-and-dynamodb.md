---
layout: post
title: "Comparing Fauna and DynamoDB"
date: 2020-12-09
description: "The truly serverless database that combines the power of a relational database with the flexibility of JSON documents."
original_url: "https://fauna.com/blog/comparing-fauna-and-dynamodb-pricing-features"
author: "Evan Weaver"
category: blog
---

Fauna vs DynamoDB


Fauna and DynamoDB are both serverless databases, but their design goals, architecture, and use cases are very different. In this post, I will overview both systems, discuss where they shine and where they don’t, and explain how various engineering and product decisions have created fundamentally different value propositions for database users.

## DynamoDB’s design philosophy: Availability and predictability

AWS DynamoDB was developed in response to the success of Apache Cassandra. The Cassandra database was originally open sourced and abandoned by Facebook in 2008. My team at Twitter contributed extensively to it alongside the team from Rackspace that eventually became DataStax.

However, in an odd twist of history, Cassandra itself was inspired by a [2007 paper](/assets/pdfs/amazon-dynamo-sosp2007.pdf) from Amazon about a different, internal database called Dynamo — an eventually-consistent key-value store that was used for high-availability shopping cart storage. Amazon cared a lot about shopping carts long before they had a web services business. Within Amazon, the Dynamo paper, and thus the roots of DynamoDB, predate any concept of offering a database product to external customers.

DynamoDB and Cassandra both focused on two things: high availability and low latency. To achieve this, their initial releases sacrificed everything else one might value from traditional operational databases like PostgreSQL: transactionality, database normalization, document modeling, indexes, foreign keys, and even a query planner. DynamoDB did improve on the original Dynamo architecture by making single-key writes serializable and dropping the baroque CRDT reconciliation scheme, and on Cassandra by having a somewhat more humane API.

## DynamoDB’s architecture

DynamoDB’s architecture essentially puts a web server in front of a collection of B-tree partitions (think [BDB](https://en.wikipedia.org/wiki/Berkeley_DB) databases) into which documents are consistently hashed. Documents are columnar, but do not have a schema.

Within a DynamoDB region, each data partition is replicated three times. Durability is guaranteed by requiring synchronous majority commits on writes. Consistency is only enforced within a single partition which, in practice, means a single document since partition boundaries can not be directly managed. Writes always go through a leader replica first; reads can come from any replica in eventually-consistent mode, or the leader replica in strongly consistent mode.

Although DynamoDB has recently added some new features like secondary indexes and multi-key transactions, their limitations reflect the iron law of DynamoDB: “everything is a table”.

* Tables, of course, are tables.
* Replication to other regions is implemented by creating additional tables that asynchronously apply changes from a per-replica, row-based changelog.
* Secondary indexes are implemented by asynchronously projecting data into additional tables — they are not serializable and not transactional.
* Transactionality is implemented via a multi-phase lock — presumably DynamoDB keeps a hidden lock table, which is directly reflected in the additional costs for transactionality. DynamoDB transactions are not ACID (they are neither isolated nor serializable) and cannot effectively substitute for relational transactions. Transaction state is not visible to replicas or even to secondary indexes within the same replica.

As you may predict from the above, the DynamoDB literature is absolutely packed with examples of “single-table design” using aggressive NoSQL-style denormalization. Using the more complex features is generally discouraged. DynamoDB’s pricing is also designed around eventually-consistent single tables, even though in replicated and indexed scenarios individual queries must often repeatedly interact with multiple tables. Furthermore, even though global tables were added a few years later, the overall pricing becomes significantly higher, while the same eventually-consistent data integrity compromise remains. designed around single-table, eventually-consistent usage, even though in replicated and indexed scenarios individual queries must interact with multiple tables, often multiple times.

Additional challenges lie in the query model itself. Unlike Fauna’s query language FQL or SQL, DynamoDB’s API does not support dependent reads or intra-query computation. In contrast, Fauna allows developers to encapsulate complex business logic in transactions without any consistency, latency, or availability penalty.

DynamoDB works best for the use cases for which it was originally designed — scenarios where data can be organized by hand to match a constrained set of predetermined query patterns; where low latency from a single region is enough; and where multi-document updates are the exception, not the rule. Examples of these restricted use cases include storing locks as a durable cache for different, less scalable database like an RDBMS, or for less-critical, transient data like the original shopping cart use case.

## Fauna’s design philosophy: A productivity journey

Fauna, on the other hand, was inspired from our experience at Twitter delivering a global real-time consumer internet service and API. Our team has extensively used and contributed to MySQL, Cassandra, Memcache, Redis, and many other popular data systems. Rather than focus on helping people optimize workloads that are already at scale, we wanted to help people develop functionality quickly and scale it easily over time.

We wanted to make it possible for any development team to iterate on their application along the journey from small to large *without* having to become database experts and spend their time on caching, denormalization, replication, architectural rewrites, and everything else that distracts from building a successful software product.

## Fauna’s architecture

To further this goal, Fauna uses a unique architecture that guarantees low latency and transactional consistency across all replicas and indexes even with global replication, and offers a unique query language that preserves key relational concepts like ACID transactions, foreign keys, unique constraints, and stored procedures, while also enabling modern non-relational concepts like document-oriented modeling and declarative procedural indexing.
If everything is a table in DynamoDB, in Fauna, everything is a transaction:

* All queries are expressed as atomic transactions.
* Transactions are made durable in a partitioned, replicated, strongly-consistent statement-based log.
* Data replicas apply transaction statements from the log in deterministic order, guaranteeing ACID properties without additional coordination.
* These properties apply to everything, including secondary indexes and other read and write transactions.
* Read-only transactions achieve lower latency than writes by skipping the log, while remaining fully consistent with additional safeguards.
          Unlike DynamoDB, Fauna shines in the same areas the SQL RDBMS does: modeling messy real-world interaction patterns that start simply but must evolve and scale over time. Unlike SQL, Fauna’s API and security model is designed for the modern era of mobile, browser, edge, and serverless applications.

Like DynamoDB, and unlike the RDBMS, Fauna transparently manages operational concerns like replication, data consistency, and high availability. However, a major difference from DynamoDB is the scalability model. DynamoDB scales by predictively splitting and merging partitions based on observed throughput and storage capacity. By definition, this works well for predictable workloads, while being less ideal for unpredictable ones, because [autoscaling changes take time](https://aws.amazon.com/blogs/database/how-amazon-dynamodb-adaptive-capacity-accommodates-uneven-data-access-patterns-or-why-what-you-know-about-dynamodb-might-be-outdated/).

Fauna, on the other hand, scales dynamically. As an API, all resources including compute and storage are potentially available to all users at any time. Similar to operating system multithreading, Fauna is continuously scheduling, running, and coorindating queries across all users of the service. Resource consumption is tracked and billed, and our team scales the capacity of each region in aggregate, not on a per-user basis.

Naturally, this design, and the related benefits, has a different cost structure than something like DynamoDB. For example, there is no way to create an unreplicated Fauna database or to disable transactions. Like DynamoDB, Fauna has metered pricing that scales with the resources your workload actually consumes. But unlike DynamoDB, you are not charged per low-level read and write operation, per replica, per index, because our base case is DynamoDB’s outlier case: the normalized, indexed data model, with the transactional, multi-region access pattern.

Higher levels of abstraction exist to deliver higher levels of productivity. Fauna offers a much higher level of abstraction than DynamoDB, and our pricing reflects that as well — it includes by default key characteristics that DynamoDB does not. At Fauna we want to provide a database with the highest possible level of abstraction that solves the use cases you would traditionally turn to a relational database for, so that you don’t have to worry about any of the low level concerns at all.

## What is an API worth?

Most other databases aside from DynamoDB and Fauna are delivered as managed cloud infrastructure, and billed on a provisioned basis that directly reflects the vendor’s costs and those costs alone. Serverless infrastructure is relatively new — S3 is perhaps the first service with a serverless billing model to reach widespread adoption — and serverless databases are even newer. The serverless model in DynamoDB is a retrofit. It is essentially still a provisioned system with the addition of predictive autoscaling.

Instead, serverlessness to date has mainly been restricted to vertically-integrated, single-purpose APIs. These APIs have been monetized indirectly like Twitter, billed per-action like Twilio, or billed as a percentage of the value exchanged via the API between third parties — like Stripe.
Serverless infrastructure, as we all know, is actually made from servers. It has a more complex accounting challenge than vertically-integrated APIs, and is constrained by:
1. Variance in resource utilization per request
2. Variance in request volume over time
3. Variance in request locality
4. Underlying static costs

The multi-tenancy of serverless infrastructure creates a fundamentally better customer experience. Who wants to pay for capacity they aren’t using? Who wants to have their application degrade because they didn’t buy enough capacity in advance? It’s also a better vendor experience, since no vendor wants to waste infrastructure, and it can be more environmentally friendly.
However, the vendor’s aggregate price across all customers must cover the static infrastructure costs, which are tightly coupled and resistant to change. (As a practical matter, a vendor can’t upgrade and downgrade CPUs, memory, disks, and networks independently of either on demand, even when using managed cloud services.) The aggregate price must also correlate with the business value recognized, and it must be appropriately apportioned based on the realization of that value for each individual customer over time.

Compared to simply marking up the incremental cost of a server, this pricing problem is hard. Let’s discuss the solutions that DynamoDB and Fauna have found.

## DynamoDB pricing

For DynamoDB’s base use case for which it was designed, its pricing is relatively clear and straightforward. However, if you add in usage of newer features like global replication, indexes, and transactions, the pricing becomes more opaque, and it can become very difficult to predict costs in advance.

## Fauna pricing

Fauna’s pricing by default maps to the underlying architectural differences we mentioned above and reflects the enhanced capability that you can tap into with Fauna.
Some key differences include:

**Compute** - DynamoDB doesn’t support computation of any kind within a query, but Fauna does. Thus, Fauna charges separately for compute costs. In DynamoDB, since computation for any particular workload can’t be done in the database at all, it must be done application-side in a compute environment like AWS Lambda, which has its own cost.

**Additional pricing dimensions** - DynamoDB has additional [pricing differences](https://www.logicata.com/blog/aws-dynamodb-pricing/) depending on specific regions, infrequent access, transaction isolation levels, and many other characteristics. It significantly complicates any cost and capacity planning exercise, which requires detailed and ongoing adjustments.

## Powering a SaaS application

A good example of how these architectural differences play out is more evident if we consider something like a typical SaaS application, for example, a CRM. We have an accounts table with 20 secondary indexes defined for all the possible sort fields (DynamoDB’s maximum — Fauna has no limit). We also have an activity table with 10 indexes, and a users table with 5 indexes. Viewing just the default account screen queries 7 indexes and 25 documents. A typical activity update transactionally updates 3 documents at a time with 10 dependency checks and modifies all 35 indexes.

And of course, we have replicated this data globally to two additional regions in DynamoDB. We will also do consistency checks on all data returned from indexes.

In Fauna, we do not need to configure replication, or do any additional consistency checking. And we benefit greatly from Fauna’s index write coalescing. Thus just the database usage costs alone with Fauna, let alone the engineering savings involved, are less expensive than DynamoDB because Fauna’s architecture was designed to support these use cases from the beginning.

Even if we assume that Fauna will require multiple write operations per document because of all the indexes, the result does not materially change. Fauna’s query pattern could also be improved by using unique constraints instead of dependency reads, which would reduce costs further.

## Using the right tool for the job

Surface level comparisons between DynamoDB and Fauna just because they are both “serverless” can result in false conclusions.

Because DynamoDB is designed for the simple use case rather than the complex, it can be more cost effective for those simple use cases. Even though core pricing for those simple use cases might be lower, you always have to consider TCO — for DynamoDB that means factoring in additional costs like manual partitioning and the eventually-consistent transactional behavior that reflect its roots as a lower-level system.

As sophistication grows — for example, if you configure a dozen indexes for a global deployment in DynamoDB — you will find your write and storage costs have multiplied by an order of magnitude compared to a single region, unindexed table. If you then change to make those writes transactional or start doing dependent reads, your costs increase even more. On the other hand, if you try to use Fauna as an application-adjacent durable cache at scale, you may find you are paying for data replication and transactional consistency that you don’t actually need.

It is more accurate to say that both DynamoDB and Fauna are great solutions that deliver on the promise of serverless when they are used for their correct purposes, and expensive when used incorrectly. This seems like a universal rule, but it actually isn’t. Most databases, even in the managed cloud, are disproportionately expensive for intermittent or variable workloads, which are prevalent with real-world workloads. This is the benefit of the serverless model: an order of magnitude less waste for both the customer and the vendor.

## Conclusion

At Fauna, we recognize that DynamoDB has pushed the envelope in distributed cloud databases and we are grateful for that. We share the same greater mission. At the same time, we know we as an industry can do better than *all* existing databases for mission-critical operational workloads, whether key-value, document-oriented, or based on SQL.I hope this post has provided you with a clearer understanding of the motivations, architectures, and business value of both DynamoDB and Fauna. And further, that this understanding helps you make more informed decisions about which tool is right for which job.