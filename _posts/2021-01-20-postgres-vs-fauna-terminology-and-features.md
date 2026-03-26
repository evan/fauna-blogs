---
layout: post
title: "Postgres vs Fauna: Terminology and features"
date: 2021-01-20
description: "The truly serverless database that combines the power of a relational database with the flexibility of JSON documents."
original_url: "https://fauna.com/blog/compare-fauna-vs-postgres"
author: "Evan Weaver"
category: blog
---

## Introduction

Postgres and Fauna are both relational, operational databases. They
serve similar use cases in modern applications, and as you will see
below, they have many similarities, especially in their positive
attributes. At the same time, they differ significantly in architecture,
query language support, and deployment model.

PostgreSQL, also known as Postgres, is one of the most popular open
source databases in the world. Derived from the Ingres project, it has
been developed by an open source community for over two decades. It is a
traditional relational database management system designed for
interactive transactions over low latency networks. Many cloud
infrastructure vendors now offer Postgres as a service, with varying
degrees of support for the full range of Postgres features.

Postgres’ architecture originated in the pre-internet era, when
operational databases were used for predictable business reporting
workloads from trusted applications within the same building on a LAN.
Various managed infrastructure providers have made proprietary changes
to Postgres’ code base to better fit the cloud world, but the
fundamentals of the architecture remain. In comparison, Fauna’s
architecture originates in the serverless era, and is designed for
scalable access from globally distributed applications over the public
internet.

Postgres and Fauna are both relational databases, supporting
serializable transactions, database normalization, foreign keys,
indexes, constraints, stored procedures, and other typical relational
features. Postgres has some features Fauna lacks, like geo-indexing,
whereas Fauna has some features Postgres lacks, like temporality,
query streaming, and multi-tenancy. Postgres offers SQL as a query
language, while Fauna supports FQL.

## Terminology

| Postgres | Fauna | Explanation |
|----------|-------|-------------|
| Row/Record | Document | An individual record in the database. |
| Table | Collection | A container for documents. |
| Primary | Region | Fauna has no primary or secondary concept, so all regions can serve both reads and writes. |
| Secondary/Replica/Standby | Region | Fauna has no primary or secondary concept, so all regions can serve both reads and writes. |
| Replication | Replication | Fauna’s replication is semi-synchronous and does not require any operator management. |
| Sharding | n/a | Fauna does not require the operator to manage sharding or partitioning in any way. |
| Primary key | Ref | The unique identifier of a document. |
| Foreign key | Ref | A pointer from one document to another. |
| Index | Index | Fauna merges the concepts of indexes and views. Indexes must be explicitly referenced, similar to the concept of hinting. |
| Transaction | Transaction | Both Postgres and Fauna support ACID transactions. |
| Schema/Database | Database | Both Postgres and Fauna have a concept of logical databases that include schema (also known as DDL) records describing the collections, indexes, and security properties of the included records. |
| Stored procedure/User-defined function | Function | Fauna supports user-defined functions written in FQL. |

## Query APIs

Postgres is based on the tabular SQL data model, while Fauna is based
on schemaless documents, to better fit modern application development
patterns. (As a historical note, initially Postgres did not support SQL,
but rather supported the QUEL language, which is similar in semantics
but incompatible in syntax. Postgres’ popularity began to grow in the
90s after the addition of SQL support. This has some similarities to how
Fauna’s popularity grew with the addition of GraphQL support.)

Postgres’ query feature set is very rich, with many custom extensions to
the SQL standard, including full-text search, the JSONB document data
type and other custom types, pgSQL stored procedures, a unique
administrative security model, and various analytical capabilities.
However, this means it is not directly compatible with any other SQL
database, which limits portability. In practice, database workloads are
never truly portable across DBMS implementations because of capability,
administrative, and performance differences.

Fauna also has a rich feature set, offering object and array types,
document streaming, temporality, attribute-based access control (ABAC),
multi-tenancy, integration with third-party identity and authorization
providers, and a standard library similar to that of a general-purpose
programming language. However, Fauna also is not directly compatible
with any other database/

Postgres’ fundamental query paradigm is that of making declarative,
analytical queries against a large dataset. However, for operational,
short-request workloads, these queries are restricted to single or small
groups of records via WHERE clauses; this, along with the tabular data
model, creates an impedance mismatch that ORMs are designed to solve.

ORMs introduce significant application complexity without removing the
need to understand the SQL model and its performance characteristics,
and undermine the value of core Postgres features like stored procedures
and views that perform complex work outside of the application layer.

In comparison, Fauna’s query paradigm is one of customizing an
operational data API via procedural programming. Fauna does not
require ORMs or other middleware, instead offering a query model that
directly matches modern application development paradigms.

You can compare SQL and FQL in detail
here.

## Indexes

Postgres and Fauna implement indexes very differently. In Postgres,
an index is a materialization of an access pattern that the query
planner can implicitly use to accelerate existing queries. In Fauna,
an index is more like a relational view: a materialization that the
application can query explicitly.

One challenge with index design in Postgres and other relational
databases is the implicit nature of their use at runtime. Frequently a
developer believes a query is efficiently indexed when it is not.
Various tools exist to try to identify unused indexes and unoptimized
queries, and even change the storage strategy the index uses, but there
is always some degree of trial and error in proper index design.

In Fauna, it is transparent when an index is queried and when it is
not, because the query must refer to it explicitly and rely on its
contents directly. This means that existing queries cannot automatically
be improved simply by adding an index. However, it also means that there
are no discontinuities in the performance profile of the query as the
dataset or query changes, simply because the planner started using a
different strategy. In relational databases, index hinting is one
technique to help mitigate this problem. You can think of Fauna
indexes as an always-hinted system.

Both Postgres and Fauna indexes can enforce unique constraints, and
both Postgres and Fauna allow indexing into deeply nested JSON
objects and multi-value arrays. Postgres supports built-in geographic
and full-text indexing, which Fauna currently lacks, although there
are some workarounds via FQL. However, unlike Postgres, Fauna indexes
can include multiple collections and create constraints across them, as
well as filter and transform indexed data in more complex ways. Fauna
can also query historical data, even in indexes.

## Schema design

Postgres and Fauna schema design is similar. Both systems encourage
relational data modeling and normalization. Both systems support foreign
keys and unique constraints. Both offer transactional guarantees that
preserve data integrity across collections and tables. Compared to
non-relational systems, data modeling in Postgres and Fauna is easy,
intuitive, and safe.

One difference is that Postgres schemas are declared and enforced at
runtime for each table, whereas Fauna is schemaless. Instead, schemas
are implied by the shape of the documents. Fauna supports more
flexible schema evolution for this reason.

To enforce specific schemas within Fauna, you can use
application-side checks, or use functions (stored procedures) for
writes, which lets the database run custom logic before transactions are
committed. Declarative schemas are on the Fauna roadmap.

Another difference is that Fauna also has graph database roots. For
example, in Postgres and other relational databases, one-to-many
relationships would be constructed via join tables. In Fauna, a join
collection works well, but you can also choose to directly refer from
one document to another via references (foreign keys) within the
documents, and index whatever query patterns you want to
pre-materialize. The index functions as a system-managed join table,
simplifying the data model.

In general, Fauna schema design can be considered an evolution of
Postgres’ relational model, with better support for modern document and
object-oriented programming practices and standards.

## Transaction model

Postgres and Fauna are both transactional databases. Postgres
supports interactive sessions transactions over TCP/IP. The application
opens a persistent connection to the database that buffers transactional
state, and can incrementally build up the transaction over the wire,
interleaving database reads, application-side computation, and database
writes.

The persistent connection model has some disadvantages in the cloud era.
It expects to interact with long-lived application instances that are
physically co-located with the database, not globally-distributed web
servers, serverless functions, or embedded clients. Connection overhead
is high, leading to cold-start problems. The series of interactions
between the client and the server require low-latency network links to
be performant. And because the server must maintain persistent resources
for each connection, connection exhaustion is possible, leading to
complex connection pooling solutions that interleave requests, affecting
consistency and availability.

Fauna, on the other hand, does not need persistent transaction state.
Fauna transactions are stateless, atomic, pure functions over the
state of the dataset. Applications using Fauna build up a transaction
locally via the Fauna drivers. This transaction is then serialized to
JSON and sent to the database as a single HTTPS request, minimizing
latency.

This also means, however, that the application cannot do local
computation during the course of the transaction—​instead it must rely on
Fauna’s rich standard library of functions for server-side
computation in the transaction itself. In practice, this is rarely a
point of friction.

This model eliminates the need for connection management. It
dramatically reduces latency, especially for geographically diverse
applications, increases throughput, and simplifies security and network
operations.

## Consistency model

On the CAP theorem continuum, Postgres and Fauna are both CP systems.
If forced to choose, they favor consistency over high availability. Both
databases offer a strictly serializable consistency model for read/write
transactions, and are ACID-compliant. Nevertheless there are some very
significant differences that make this comparison not as straightforward
as it may seem.

Postgres defaults to the read-committed isolation level, which is much
weaker than serializable. Enabling serializable isolation has some
performance costs. Additionally, Postgres only offers transactional
isolation for transactions that interact with the primary node.

Most applications perform all read-write transactions on the Postgres
primary, but route read-only transactions to secondaries to allow for
read scale-out and lower latency. These read-only transactions are
eventually consistent. The results can be stale or even invalid under
common scenarios without any notice to the application.
Read-your-own-writes is not guaranteed. Even when Postgres is configured
for semi-synchronous replication, this situation does not improve.
Postgres will block writes on secondary commits, but will not isolate
secondary reads.

Enabling multi-primary writes in Postgres degrades the isolation model
even further, as constraints can no longer be enforced. Avoiding these
problems requires using read replicas as hot standbys only, leading to
the obvious scalability bottlenecks on the primary node. Postgres’ roots
as a single location, vertically scaled RDBMS are clear when the
isolation model is combined with replication.

In contrast, Fauna defaults to strict serializability, the highest
possible isolation level. This guarantee is preserved for all read-write
transactions, regardless of which region they are performed in. This
reduces latency and increases durability. Unlike Postgres, there are no
circumstances in which Fauna can lose an acknowledged commit.

Fauna read-only transactions default to snapshot isolation, which is
similar to Postgres’ read-committed default for all transactions, and
are always served from the closest region to the application without
further coordination. This minimizes latency. However, Fauna also
uses some special techniques that upgrade read-only snapshot isolation
to strictly serializable in all normally observable scenarios, without
increasing latency.

Thus, Fauna’s consistency model is stronger than Postgres’ model for
horizontally scalable and/or highly available deployments.

## Security

Postgres has a complex security model that has grown in complexity over
time. For basic access, it offers the ability to inherit security
properties and accounts from the underlying operating system, limiting
access to datasets and administrative resources. It allows for creation
of administrative accounts directly within the database itself. It
offers connections over TCP/IP, with optional encryption via SSL, as
well as connections via Unix domain sockets.

Once access is achieved, various administrative rights can be granted
for table access and schema changes. Additionally, row-level security is
possible by creating policy predicates that are checked before
performing read or write operations.

Postgres security is designed for a small number of administrators of
the database itself. It is not designed to manage security at the
application user level. This task is completely delegated to the
application itself. The developer must decorate every query with
appropriate access restrictions, and consider the implications of
indirect access and other leak vectors in the dataset. This can lead to
many security faults, such as the venerable SQL injection attack, where
malformed user data can truncate a query before its security clauses and
defeat them.

Fauna, on the other hand, was designed with a web-native security
model. It is secure by default and does not assume that any access is
trusted. All applications must connect to the database over HTTPS with
an access key that defines the administrative privileges that
application has.

Additionally, Fauna’s security model additionally assumes user-level
security is the common case. Users can be authenticated and identified
with third-party providers or with Fauna directly, and be issued a
key that limits their access, which is controlled by attribute-based
access control policies, which are similar to Postgres’ predicates, but
more flexible and composable.

Fauna also offers a hierarchical tenancy model that lets datasets
within the same database be completely isolated from each other, for
example, different customers of a SaaS product. Such a data model in
Postgres requires the addition of a tenancy column to every table and a
filter clause to every query, introducing complexity and risk.

Additionally, query parsing in Fauna is type safe, and injection
attacks are not possible.

## Replication and high availability

Postgres’ replication is based on a traditional primary and secondary
concept. All write transactions must be directed to the primary node.
After they commit on the primary node, the write effects are
asynchronously replicated to any secondary nodes. (Some previous
versions of Postgres required statement-based replication based on
triggers and middleware.)

The Postgres replication architecture introduces several issues in a
modern cloud environment. The reliance on a primary node creates a
single point of failure, reducing availability. It also increases
latency because it can not be globally distributed. It is a scalability
bottleneck, requiring vertical (per table) partitioning as a workaround,
which undermines transactional guarantees. Serializability and
read-your-own-writes guarantees are not met when reading from the
secondary nodes. Finally, if the primary node fails, it is possible to
lose committed writes, as well as suffer a lengthy failover process
during which writes are non-performant.

Fauna’s replication is based on a Calvin-inspired, global, replicated
transaction log. All transactions are journaled to the log before they
are committed. Commits to the replica regions are semi-synchronous; the
application does not receive an acknowledgement until durability,
serializability, and read-your-own-write guarantees are met. This
applies regardless of which region the application accesses, and even if
the application switches regions between requests.

There is no concept of a primary or secondary region in Fauna, nor is
there a user-visible concept of a node. The Fauna API manages
failures of the underlying infrastructure transparently for both reads
and writes, and latency is bound by the physical distance to the nearest
Fauna region.

## Scalability

Postgres scalability is bound on two axes: the capacity of the primary
node, which principally impacts write transitions (and the read
dependencies of those writes), and the number and capacity of the
secondary nodes, which can scale reads. Thus, writes are vertically
scalable, and reads are both horizontally and vertically scalable. In
order to scale in most Postgres clusters, manual provisioning is
required, or at minimum, configuration of auto-scaling parameters.

Postgres is not, at heart, an elastic system. Scaling up and down must
be a predictive process, because it takes time to shutdown and restart
processes on new hardware, VMs, or in new containers, rehydrate caches,
and copy data from secondary nodes. Cloud Postgres systems that are
serverless or auto-scaled typically rely on making many
micro-provisioning decisions on an hourly or minutely basis to simulate
an API-style elasticity. This impacts latency, throughput, and
potentially availability, depending on configuration. It also leads to
wasted, idle resources.

Fauna’s writes scale with the transaction log, which is both
partitioned and replicated, and thus horizontally and vertically
scalable. Fauna’s reads scale with the number and size of the replica
regions, which may be heterogeneous given the balance of workloads
within Fauna overall. However, these scaling decisions are not
visible to the developer at all.

Fundamentally, Fauna is a multi-tenant API, not a provisioned,
managed resource. All workloads within Fauna share a large pool of
underlying hardware that is both multi-region and multi-cloud. The
Fauna scheduler within the database kernel tracks resource
consumption and implements something akin to cooperative multithreading
to isolate queries from each other.

Thus there is no concept of scaling a database up and down within
Fauna. All workloads potentially have access to the resources of the
entire cluster at any moment in time. The Fauna operational team
scales the shared Fauna clusters behind the scenes as overall usage
grows. This model minimizes idle resources and waste, and helps
guarantee predictable performance regardless of the individual query
patterns within a database.

## Deployment

Postgres is open source, and relies on operator deployment onto
provisioned hardware. Many cloud vendors offer managed versions of
Postgres that orchestrate this deployment via a dashboard or
administrative API.

This model exposes quite a bit of complexity, but also choice, to the
operator. The operator controls which version of Postgres is deployed,
coupling operational and performance improvements, bug fixes, and new
features together. The operator must also select the underlying hardware
for the database, which can dramatically change the performance not only
of different types of queries, but also the running costs of the system.

Postgres has a substantial library of first-party and third-party
extensions that may or may not be available or configurable. Tuning
parameters that are available for things like buffer caches, connection
defaults, background tasks behavior (such as the vacuum task that
garbage collects deleted records) all affect performance and
configuration. Additionally, Postgres failover models vary and require
understanding of the host environment’s capabilities and the database’s
configuration policies in order to manage hardware faults appropriately
from the application.

In contrast, Fauna abstracts these decisions away. The Fauna
operations team manages the cluster overall, continually upgrading the
underlying software and hardware to improve performance across the board
for all Fauna databases. Fauna databases have no configurability
at the operational level beyond region selection, and performance is
bound by query patterns and dataset sizes, not by tunable parameters.

Open source has many benefits, but in this case, since Fauna is
proprietary, one of its downsides is avoided. There is no feature
fragmentation: every Fauna database has access to the same
capabilities, including those in the local development edition (which is
a copy of the actual cloud software, and not a mock). There is no
upgrade cycle or version selection for the operator; compatibility is
maintained through API versioning instead.

## Jepsen tests

Jepsen, and its related tool Elle, is a verification suite for the
consistency properties of transactional systems, under normal operation,
and under fault conditions. Both Fauna and Postgres have been
evaluated under Jepsen at various times in the past.

In both evaluations, the Jepsen team found implementation bugs that
affected the consistency properties, which have since been resolved by
respective maintainers of the databases. In many Jepsen analyses, the
underlying system cannot possibly deliver its claimed capabilities due
to fundamental issues in the architectures. Importantly, the Jepsen team
found that both Fauna’s and Postgres’ architectures achieve their
goals in both theory and practice.

One important thing to note is that a Jepsen analysis only checks for
the properties that the system under testing claims to offer—​it is not
testing against an absolute standard. Since Postgres does not offer
multi-primary write consistency, 100% durability on failover, or
serializable reads from secondary nodes, these properties were not
checked.

Fauna does offer these properties, and they were verified as such.
Thus, Jepsen ultimately demonstrated that although Fauna is newer
than Postgres, it does indeed offer an overall higher level of
consistency.

## Summary

Fauna and Postgres share many similar goals and characteristics.
Although Postgres is older than Fauna, they are both tested and
hardened transactional databases with very strong durability,
consistency, and availability properties. Their differences are most
apparent in two ways: their query languages and their operational
posture.

Postgres is the archetypal RDBMS, like DB2 or Oracle. It is designed for
high-availability, mainframe class hardware, with co-located, trusted
applications making SQL queries on a low-latency LAN.

Fauna, on the other hand, is two or even three generations advanced.
Designed recently rather than in the 80s and 90s, it reflects modern
development and deployment patterns: globally distributed applications
that use document and graph APIs to securely access data over the public
internet.

If you like Postgres, especially its transactional properties and
battle-hardened reliability, but want to minimize your operational
burden and maximize your productivity in the modern cloud and serverless
era, you may like Fauna even more. Give it a try!