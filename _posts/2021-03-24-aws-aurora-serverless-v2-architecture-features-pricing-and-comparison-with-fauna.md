---
layout: post
title: "AWS Aurora Serverless v2: Architecture, Features, Pricing, and Comparison with Fauna"
date: 2021-03-24
description: "Amazon recently announced the preview release of AWS Aurora Serverless v2. Aurora and Fauna are both serverless databases. Let's spend some time comparing Aurora's architecture, features and pricing w"
original_url: "https://fauna.com/blog/compare-aws-aurora-serverless-v2-architecture-features-pricing-vs-fauna"
author: "Evan Weaver"
category: blog
---

*This article is based on publicly available information only and was updated on May 20th, 2022 to address the general availability of Aurora Serverless v2.*

At the all-virtual Re:Invent 2020, Amazon announced the preview release of AWS Aurora Serverless v2, a relational database-as-a-service offering MySQL and PostgreSQL interfaces. Although Aurora Serverless v1 was only really useful for testing, Aurora Serverless v2 is intended for production workloads.

Aurora Serverless v2 reached general availability (GA) on April 1st, 2022. However, Aurora Serverless v2 is of limited use for production workloads, particularly for serverless applications. The lack of support for infrastructure as code (IaC) support via AWS CloudFormation prevents users from creating and tearing down environments in CI/CD pipelines. There is no support for scale to zero; users can provision down to one half an Aurora capacity unit (ACU) at a cost of $43.20 per month in the *us-east-1* region.
Aurora Serverless v2 does not support the popular Data API, requiring users to establish, maintain, and manage persistent connections to the instance.

Although Aurora and Fauna are both serverless databases, comparing them is a bit of an apples vs. oranges situation. Aurora is architecturally rooted in applying autoscaling to managed virtual machines. Despite the more granular provisioning and billing in Aurora Serverless v2, many legacy operational concerns remain. However, Aurora’s architecture allows it to maintain compatibility with existing PostgreSQL and MySQL workloads — a major benefit for developers who need to migrate from on-premises to cloud.

Fauna, on the other hand, is architected as a global data API. It is designed to eliminate operations entirely from the developer experience of using a database.

Since Fauna is an API, and not a managed service, developers using Fauna do not need to configure consistency, locality, capacity, or make provisioning or versioning decisions. Fauna's developer experience is web-native and serverless and fits modern application development patterns without problems with connection management, impedance mismatches in security models, cold start latency, or caching.

Additionally, Fauna is a programmable database, borrowing some concepts from previous programmable datastores like [Linda](https://en.wikipedia.org/wiki/Linda_(coordination_language)). The ability to run compute transactionally within the data API allows for flexible and maintainable business logic composition without additional services or the methodological problems of the stored procedure model.

I would like to explore the reasoning and implications of these design decisions in this post, and spend some time comparing Aurora Serverless v2 with the current version of Fauna.

## Aurora’s design philosophy

AWS’s first managed database option was RDS (the creatively named “Relational Database Service”). RDS, which offers MySQL, Postgres, Oracle, and SQL Server, is a typical managed service. From a performance, availability, and scalability perspective there is no fundamental difference between RDS and what you would get if you deployed a relational database on an EC2 instance with local block storage.

RDS offloaded much of the administrative burden of operating an RDBMS while maintaining legacy compatibility, but many problems remained. Vertical scalability was ok, but came at the cost of high availability. Upgrading, migrating, or restarting the database process or node would cause an extended outage as the caches had to reheat. Durability was at risk since the primary node had only a single copy of the dataset. Also, IO throughput was limited by the underlying local or EBS storage, restricting performance regardless of the number of cores dedicated to the database machine.

How could RDS availability and performance be improved? AWS had the insight [that](https://www.percona.com/blog/2016/05/26/aws-aurora-benchmarking-part-2/) open source database software could be modified to cooperate with multi-tenant direct-attached storage, instead of using a dedicated disk. A multi-tenant SAN would improve availability and performance by replicating the data multiple times behind a logical interface, and also by sharing the aggregated pool of IO resources across tenants, who presumably do not need to access it all at the same time.

This shared-disk architecture, instead of the shared-nothing architecture of standard RDS, is the core of Aurora--both the provisioned and serverless edition. Since the database software must be modified to take advantage of the altered architecture, only the open-source MySQL and PostgreSQL interfaces are available.

This means that Aurora is a great choice for existing applications that already use PostgreSQL or MySQL and are trying to migrate to the cloud. Many of these applications were written in the last 20 years using object-oriented frameworks like J2EE, Ruby on Rails, PHP, etc. Although there will be some performance and feature discontinuities, Aurora is committed to compatibility with these existing interfaces so the migration lift is relatively light, for the benefit of reduced operational overhead and increased infrastructure flexibility (but not necessarily decreased costs, depending on workload and existing infrastructure investment).

This compatibility commitment, however, also means Aurora works best with three-tier application architectures. The database expects to be accessed from trusted, long-lived clients within the same security perimeter. Using serverless technology creates an impedance mismatch. For example, connections still have cold start overhead, making querying from AWS Lambda or other serverless compute infrastructure more latent. To secure the database, a VPC is required, or middleware like Aurora Data API. This adds complexity and reduces performance.

The shared codebase also restricts Aurora’s consistency model to primary/secondary replication. It is impossible to perform write transactions or consistent read-only queries in non-primary processes and regions. This means that writes and consistent reads from non-primary regions have dramatically increased latency compared to systems with fully-distributed transaction algorithms. And because query consistency depends on which specific region and process is accessed, the application must be topology-aware. This creates a number of additional, undifferentiated operational burdens for the developer, and it’s easy to get wrong and introduce subtle data corruption.

With these goals in mind, let's explore how Aurora is implemented.

*Update: PostgreSQL is now available in Aurora Serverless v2.*

## Aurora’s architecture

Aurora has three architectural components: a routing layer, a query layer, and a storage layer. All three services are effectively proprietary, even though the query layer is based on open-source code.

### Routing layer

The routing layer accepts database connections from clients and multiplexes them onto connections to the query layer processes. It is horizontally scalable and multi-tenant. The routing layer is protocol-aware, and conceptually similar to PgBouncer or other connection pooling solutions, although with Aurora-specific optimizations. It supports the MySQL and PostgreSQL wire protocols.

Aurora’s routing layer serves two main purposes beyond simply routing requests to the right processes. It helps with scalability by offloading connection overhead from the database processes themselves. It also allows Aurora to hold client connections open even as the database processes are being migrated or restarted.

The routing layer also tracks usage of the underlying database connections in order to inform auto-scaling, which is important for the serverless model as we will see later. Some versions of Aurora also support integration with a SQL-over-HTTP service called Data API, which is an additional service beyond the routing layer. However, as of 19 May 2022, Aurora Serverless v2 does not support the Data API.

### Query layer

The query layer, or database layer, is the most traditional component of the stack. It connects to the routing layer, executes the computational components of the queries that clients submit, and accesses the storage layer to read and write the underlying rows and indexes.

The query layer "editions" are forks of the underlying open source PostgreSQL and MySQL database software. They are so closely tied to the original code that the operator must select a specific database minor version when setting up the cluster. Typically, with software-as-a-service, a single API service supports multiple versions of the interface, but Aurora is actually running a specific version of the database process unique to each customer cluster. It is not offering a multi-tenant API. Auto-scaling and auto-failover aside, this is effectively the same experience as hosting the open source database code yourself.

Within an Aurora cluster, there can be many read-only database processes, but only one writer--no different than a primary/secondary RDS configuration.

Since database processes are not multi-tenant, but rather provisioned instances of virtual machines running the customer-selected process, reducing provisioning granularity and increasing provisioning speed becomes very important for the serverless model, as we will see later on.

### Storage layer

In Aurora, the storage layer manages replication both within regions and across regions. This storage-level replication differs from standard RDS and most self-hosted RDBMSes, which manage replication at the query layer.

As a practical matter, read-write transactions, or strongly consistent reads, must be routed to the single primary process, no matter what region it is in. Every read from any other process, including region-local processes, is eventually consistent. (It is possible to configure Aurora MySQL in a multi-primary topology, which introduces some degree of query layer replication coordination, but this reduces the consistency guarantees available and impacts latency in unpredictable ways, and still is limited to a single write region.)

Because of this constraint, in Aurora, read-write transaction latency varies proportionally with the physical distance between the client and the primary process. Aurora has a few tricks like process-to-process forwarding of write effects within region-local replicas that try to reduce replication latency, but it cannot avoid the fundamental architectural need to route all writes, and all consistent reads, through the primary process. This additionally can create contention within that process as concurrent read-write transactions have no isolation from each other.

To provide high availability, every Aurora region keeps six copies of the dataset: two copies in each of three availability zones (AZs). This lets it tolerate a series of specific failures while minimizing application impact:

* Loss of two partition copies in different AZs does not have user-facing impact
* Loss of two partition copies in the same AZ will affect latency and may require a primary process failover to another AZ to mitigate, but the database will remain available for writes
* Loss of three partition copies will make the database unavailable for writes, but will not cause data loss
* Loss of four or more partition copies makes the database completely unavailable and may lose data

The Aurora storage layer also manages backup, restore, and other disaster recovery scenarios, simplifying the query layer. Aurora uses a log-structured storage engine, unlike the B-tree engines that PostgreSQL and MySQL use by default.

### Aurora's original scaling model

The original Aurora query layer was manually provisioned: the operator could configure a primary database VM in a series of standard sizes, as well as secondary read replica VMs. Instance sizes were offered in powers of two, like EC2. Since restarting the database process on a new VM flushes the buffer pools (caches) which are critical for maintaining performance for the working set in an RDBMS, resizing the VMs interrupted availability.

It was possible to configure the AWS auto-scaling tools to watch metrics and resize an Aurora cluster automatically, but due to cold start issues and the general mission-critical nature of the operational workloads typical for an RDBMS, this was not very useful.

AWS made a number of attempts to improve this situation. First, they added an algorithm that pre-populates the buffer pools of new VMs based on the contents of the storage-level page cache--a big improvement over the hours it could take for the buffer pool to warm “naturally”--but the cost of the cold starts remained significant.

Next, Aurora Serverless v1 attempted to solve this by keeping a bunch of reserve VMs running, and copying the actual contents of the buffer pool [across the network](https://www.slideshare.net/AmazonWebServices/going-deep-on-amazon-aurora-serverless-dat427r1-aws-reinvent-2018) from the old VM to the new one on scaling events. However, even an in-memory copy across the network takes time at large heap sizes. This is one of the reasons that Aurora Serverless v1 was [ultimately only recommended](https://dev.to/dvddpl/how-to-deal-with-aurora-serverless-coldstarts-ml0) for non-production workloads. Additional Aurora features like multi-region replication were never implemented in Aurora Serverless v1.

Now, Aurora Serverless v2 has entered the picture as the third effort to address Aurora autoscaling.

### Scalability improvements in Aurora Serverless v2

The goal of Aurora Serverless v2 is to make Aurora Serverless production-ready, by expanding the data recovery and replication capabilities to match provisioned Aurora while simultaneously making auto-scaling truly seemless.

Although it’s straightforward to track compute and connection utilization in order to create an auto-scaling policy, the scaling events themselves must avoid:

* Buffer pool flushes and copies
* Connection resets
* Cancellation of long-running queries

Also, ideally the database must be able to completely quiesce (scale to zero), and consume no resources when there are no active queries--while still holding connections open.

AWS appears to have solved this by moving from an arms-length vertical scaling model (shutting down the VM and creating a new one with different provisioning settings) to a co-operative one. The buffer pool is stored in a separate process, and the VM hypervisor can now dynamically add resources to the running VM and notify the database process and buffer pool process that it is OK to consume more resources. On scale down, it notifies the processes to reduce their utilization, then removes the resources from the running VM. Thus, more capacity is made available without moving the workload across the network (and possibly without restarting the process at all, although that is unclear to me).

All versions of Aurora are deployed onto VMs from AWS’s memory-optimized database instance classes. These instances have one hyperthread per 4GB of memory (presumably CPU bursting is also enabled for the smallest VM sizes). Aurora capacity is measured in “ACUs”, which are equivalent to half a hardware thread (hyperthread, not a core) and 2GB of memory.

Unlike provisioned Aurora, and Aurora Serverless v1, which both scale up and down by 2x existing capacity, Aurora Serverless v2 can scale in half-ACU increments (1GB memory and 1/4 thread at a time).

You will note, however, that the new scaling model requires AWS to maintain idle resources not just in a group of pre-started VMs that can be provisioned from the general EC2 fleet, but *on the machine*. It appears, based on the pricing, that AWS is targeting 50% utilization per machine, which would work well for most small serverless clusters that need to burst periodically. Rebalancing tenants across machines to achieve the per-machine utilization goal over time most likely uses the same cache-copying process as Aurora Serverless v1.

Additionally, Aurora can scale read-only workloads horizontally by automatically provisioning more VMs. Quiescing readers and the writer process is still subject to the “scaling points” restriction from Aurora Serverless v1: in particular, long-running queries and some background tasks like index construction cannot be moved, so the process and its associated VM must hang around until they are complete. VM migration to handle tenant rebalancing is subject to the same restrictions. Moving beyond the process model for serverless would require a completely new architecture.

*NOTE: Although the ability to quiesce without cold start problems is a requirement for serverlessness, it’s not clear that Aurora can actually do this. Aurora Serverless v1 offers a “sleep” mode that will enable quiescence when there are no connections open to the database--but waking up to process even a single query takes approximately 30 seconds. Aurora Serverless v2 preview currently offers no sleep mode at all, requiring a minimum of 4 ACUs running at all times.*

## Fauna’s architecture

In some ways all database management systems are the same. Fauna has a similar internal organization to Aurora, and like Aurora, Fauna’s internal services are proprietary.

However, Fauna has five big architectural differences from Aurora and from other RDBMSes:

1. Fauna has a separate transaction log service, which ensures data consistency while maintaining horizontal scalability and high availability
2. Fauna is programmable, with a rich standard library of functions. It can run transactional business computation adjacent to the data and let the developer compose computational logic in a maintainable way
3. Fauna has a dynamic multi-tenancy model, instead of process-based provisioning
4. Fauna uses stateless, secure HTTP connections directly, instead of via translation
5. Fauna is natively temporal, which means it tracks document and schema history and makes it available for querying within the query languages themselves

These differences are important for fit-and-finish in a serverless world, as we will see later on.

### Routing layer

Fauna’s routing layer acts like a standard web server. Unlike Aurora, PostgreSQL, and MySQL, Fauna queries are accepted over HTTPS, not a binary wire protocol, and each query is self-contained and does not rely on connection state beyond request headers. This means that Fauna works well with standard HTTP proxy and caching stacks, firewalls, browsers, and mobile apps.

Unlike Aurora, which is designed to be accessed within a trusted security perimeter, Fauna’s security model is designed for the public internet, like a typical API. All the query interfaces Fauna supports are designed to fit this web-oriented model. They are expressed as pure functions over the state of the database at a point in time, which matches Fauna’s consistency model too. Every request to Fauna executes transactionally, regardless of which region the request is processed in.

Once an HTTP request has been accepted and parsed, Fauna then dispatches the query AST to the local query layer via an internal interface.

### Query layer

The query layer in Fauna executes the query--except for writes. It interprets the query AST and performs pure compute operations in-process and read operations by querying the storage layer. It also tracks proposed write effects in order to construct the intermediate representation of the query to submit to the log.

The dynamic multi-tenant isolation Fauna provides is primarily managed by the query layer, although the log must manage it too. In the query layer, priority levels are consulted and queries are concurrently scheduled, parked, and rescheduled until they complete.

MySQL and PostgreSQL both use conventional threading models for their concurrency abstraction. MySQL uses POSIX threads, and PostgreSQL uses a child process-per-connection. This means that query scheduling is controlled by the operating system, not within the database’s query layer. Aurora inherits these models--it has no ability to limit resource consumption by query, guarantee fair scheduling between concurrent queries, or reliably interrupt long-running queries.

Fauna, on the other hand, does not dedicate operating system resources per query or per connection. Instead, queries are cooperatively multi-threaded in userspace code. Their execution is interleaved on threads within a shared threadpool, but function branches and IO requests within the query act as yield points so the query layer can check resource consumption and relative priority and make very granular rescheduling and pausing decisions. This allows for safe, high-level management of concurrency even when multiple tenants are competing for the same resources.

In Fauna, after executing the query, if the query contains writes, it must be submitted to the next component we will look at: the log.

### Log layer

Fauna has one component that Aurora does not have: the transaction log. This service, inspired by the [Calvin paper](/assets/pdfs/calvin-sigmod12.pdf), guarantees strict serializability for read-write transactions. It also serves as a logical clock for all transactions, eliminating the need to rely on atomic clock synchronization. Because of the transaction log, Fauna can offer read-only serializability without cross-datacenter coordination.

Also because of the log, Fauna has no concept of a primary region. Fauna’s replication works semi-synchronously, replicating read-write transactions in the log before any data in the replicas is actually modified. This lets Fauna commit transactions with only a single global round-trip of latency.

The log is logically global, but physically partitioned and replicated across multiple datacenters in the Fauna service. This means that Fauna can offer scalability and performance capabilities that Aurora lacks, specifically:

* Horizontal scalability for writes
* A predictable latency profile for writes (in a global configuration, less than 150ms)
* Very low latency for consistent reads (less than 10ms regardless of topology), eliminating the need to worry about eventual consistency

To learn more about the log protocol itself, please refer to our previous article.

Once transactions are durably replicated on the log, they are then forwarded to the storage replicas.

### Storage layer

Fauna's storage replicas listen to the log in order to update their consistent view of their data partitions, and also serve read operations to the query layer. Fauna’s storage engine is relatively simple, because many of the traditional database storage concerns like indexing, concurrency control, and replication are instead managed by the log and the query layer in Fauna.

This is different than Aurora, which has altered the traditional database architecture in the opposite way. In Aurora, replication and concurrency are storage layer concerns, not query layer concerns. This increases the complexity and decreases the flexibility of the storage options in Aurora. For example, every Aurora cluster is locked to 6-way replication per region.

Fauna’s storage layer implements a wide-row hash-partitioned log-structured merge tree, similar to LevelDB or RocksDB. The individual storage files are immutable, which is useful for a variety of reasons, but also require periodic compaction to maintain read performance.

Fauna currently uses node-local direct attached storage, which can be scaled semi-independently from compute and memory, depending on the underlying cloud infrastructure. Again, unlike Aurora, Fauna storage is not a separate service with its own distributed coordination scheme.

## Serverlessness and query isolation

As mentioned previously, the query isolation model (in a performance sense, not a consistency sense) is one of the bigger things that differentiates the serverless experience between Fauna and Aurora.

### In Fauna

Fauna is dynamically isolated, like an API--because it is an API. No resources are ever statically provisioned on a per-tenant basis, and noisy neighbor issues are handled with the query layer scheduler, instead of by the operating system or via VM isolation. This means:

* There is no minimum capacity requirement per database
* Databases that aren’t being actively queried consume no resources, aside from storage
* There is no maximum capacity limit per database
* Every database is always highly available in every region, regardless of previous query locality
* The developer is freed from having to manage capacity and locality

Since Fauna shares memory and compute across tenants, any process can answer any query. The entire capacity of the cluster is scheduled across active concurrent queries on an effectively instantaneous basis. There are no cold starts, scale up/scale down delays, or minimum runtime requirements. This allows Fauna to bill on a more predictable per-operation basis, instead of a time basis, as well.

### In Aurora Serverless v2

Unlike Fauna, Aurora Serverless v2 is still a partially-provisioned system. Tenants are isolated not by multiplexing queries on a per-operation basis, but instead by making provisioning decisions at the virtual machine level, so that tenants literally have separate database servers.

Aurora cannot react to the immediately observed query demands. Instead, it must measure VM utilization over time and ensure that future queries will have resources available to execute. It does this by tracking CPU and connection utilization as an indication of capacity used, and provisions more resources when available capacity drops below the target. When utilization drops, it tries to decommission the excess capacity in order to maintain the target values.

This model creates a number of efficiency issues:

* The database cannot effectively quiesce, because starting a new database process takes approximately 30 seconds, timing out any requests issued in the process. The alternative, to keep an idle process running, wastes resources.

* Scaling a process up is fast, as long as there are machine-local resources that can be added to the process’s host VM. But scaling down requires releasing memory, connections, and threads. In particular, releasing memory requires dropping part of the cache.

* Similar to scaling up, scaling out by adding new read processes on new VMs is straightforward, barring cold start time, but scaling back in may be blocked until long-running queries complete.

* Because scale-down is computationally costly, Aurora limits scaledown events to approximately every 30 seconds. This means that a single query can incur 30 seconds of provisioned time regardless of need, and a burst of queries could push the vertical scale high enough that many 30 second cooldown cycles are required to bring capacity back to baseline. Similar to the scale-to-zero issue, this can waste significant resources.

* The dimensions of capacity are not continuous. Query performance varies as scale changes due to the combination of vertical and horizontal scaling across individual read and write processes.

All transactional cloud databases to date have roughly followed this model and been based on some version of autoscaled, provisioned physical isolation. They all suffer from the same scalability constraints. Although some NoSQL solutions like Azure CosmosDB, AWS DynamoDB, and Google Firebase are API-driven, it comes at the cost of basic database functionality like generalized ACID transactions, unique constraints, and other basic relational features. As far as we know, Fauna is the only transactional cloud database that offers a dynamic tenancy model.

## Multi-tenancy and operational isolation

Although Aurora enforces tenant isolation across different AWS accounts, it offers nothing for isolation within databases--for example, for SaaS workloads where data is never shared across customers. The usual SQL techniques would be to add a “tenant” column to every table. This strategy is prone to security holes and noisy neighbor problems. Also, the query optimizer is not tenant-aware and data is not clustered by tenant.

Instead, Aurora [suggest](https://severalnines.com/database-blog/benchmarking-managed-postgresql-cloud-solutions-part-two-amazon-rds)s in this scenario that a new Aurora database be provisioned for every customer. This suggestion seems misguided to me:

* It requires using the AWS API from within the application to create new customer databases, instead of SQL, leading to a host of new security concerns
* It adds an additional dimension of physical isolation that interacts very poorly with Aurora’s scalability model

Consider a typical SaaS application, like a CRM. Most businesses using the CRM are small, and are not actively entering or querying data all the time. Instead they have intermittent workloads. However, because of Aurora’s cold start latency, idle capacity must be provisioned per-database to guarantee good performance. Even if the cold start latency is considered acceptable, every burst of queries requires a minimum 30 seconds of provisioned database time--for a web or API request that may have lasted no more than 100ms.

This strategy exposes the developer to pathologic cost scenarios when many low scale databases are accessed infrequently. Following the Aurora recommendations would lead to a minimum marginal cost of over $1,050 per customer to achieve performance and security isolation at the database level.

In Aurora, any additional dimension on which the developer needs isolation compounds costs disproportionately by spreading load across more VMs. This includes multi-region deployments, which are physically isolated by definition.

In Fauna, however, this strategy works. Databases can contain other databases recursively, like a filesystem, and as we saw previously, consume no resources unless used. (Also, Fauna plans are charged per account, not per database; you can have an unlimited number of databases within just one account.)

## Comparing Fauna’s developer experience with Aurora

As we have now seen, Fauna and Aurora have very different architectures. On top of the differences from standard PostgreSQL
or MySQL, these differences lead to a different developer experiences. Operationally, Fauna offers less, in the good sense of less:

* Fauna doesn’t require connection pooling
* Fauna doesn’t require cache management
* Fauna doesn’t require capacity planning
* Fauna doesn’t require a minimum capacity to avoid cold start latency
* Fauna doesn’t require a special security perimeter--it is secure for access over the public internet, and integrates with [third-party](https://forums.aws.amazon.com/thread.jspa?threadID=312199) web identity providers

From a features perspective, Fauna offers a variety of unique capabilities that fit well into serverless stacks:

* Fauna’s FQL  API fits modern application development patterns without the use of middleware
* Fauna is always transactional and consistent, no matter where and how it is queried
* Fauna offers streaming built-in, not through a second system
* Fauna offers database management via FQL, not via a separate operational API
* Fauna has built-in temporality
* Fauna supports the database-per-customer pattern for SaaS applications

Fauna’s pricing is based on operations performed, while Aurora's is based on process running time. Let’s explore that point.

## Aurora serverless pricing, explained

Unlike Fauna or DynamoDB, it is not possible to reason about Aurora Serverless v2 pricing on a per-operation or per-query basis. Instead, one must consider the cost per time period and measure how much work a single time period can do.

### Minimum costs

Aurora Serverless v2 ACUs cost $0.12 per ACU-hour, [twice as much as provisioned Aurora ACUs](https://www.jeremydaly.com/aurora-serverless-v2-preview/#:~:text=Instead%20of%20needing%20to%20attach,milliseconds%20based%20on%20application%20load.&text=Also%2C%20because%20the%20instance%20scaling,ACUs%2C%20you%20get%2018%20ACUs.). This means that:

* The minimum current running cost is 4 ACUs: $0.48 per hour or $350 per month
* The minimum scalability increment is 30 seconds of runtime for a half-ACU, or $0.0005 (this means there is a possible worst-case cost of $500 per million transactions for workloads that pathologically trigger the auto-scaling function)

The baseline Aurora Serverless V2 cost is thus $350 monthly. Each auto-scale event will cost a minimum of $0.0005. The equivalent capacity in provisioned Aurora is $175 monthly, but that would lack the responsive serverless auto-scaling.

Presumably AWS will reduce the minimum running cost over time, but they cannot eliminate it entirely due to the cold start problems with the process-based architecture. Two ACUs, the minimum capacity of provisioned mode, would still cost $175 monthly. This cost also does not include a number of things like primary and backtrack storage, read replica processes, multi-region replication, and bandwidth.

### Compute capacity

Now we need to figure out how much work one time period can do. AWS has published some [benchmarks](https://severalnines.com/database-blog/benchmarking-managed-postgresql-cloud-solutions-part-one-amazon-aurora) for Aurora in provisioned mode, but some of them trend towards [benchmarketing](/assets/pdfs/aurora-benchmarking.pdf): for example, reading a single column, from a single row, from a dataset that fits entirely in a small amount of RAM on a monster 64-core MySQL node.

Instead, we can turn to *pgbench*, the built-in PostgreSQL benchmarking tool. By default, it runs a simple, yet realistic write-biased transaction based on TPC-B. It also includes transactional dependencies that require concurrency control. Notably, it does not include indexes, but relies exclusively on primary key lookups. Here is the actual transaction:

```
    BEGIN;
    UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
    SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
    UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
    UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
    INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
    END;
```

This transaction contains 3 logical read operations and 4 logical write operations. In PostgreSQL, this will run at the read committed isolation level by default.

In Fauna, this is equivalent to:

```
Let({
  account: Get(Ref(Collection('accounts'), '13')),
  teller: Get(Ref(Collection('tellers'), '19')),
  branch: Get(Ref(Collection('branches'), '17'))
},
  Do(
    Update(Select(['ref'], Var('account')), Add(Select(['balance'], Var('account')), 500)),
    Update(Select(['ref'], Var('account')), Add(Select(['balance'], Var('teller')), 500)),
    Update(Select(['ref'], Var('account')), Add(Select(['balance'], Var('branch')), 500)),
    Create(Collection('history'), { data: { 
        aid: Ref(Collection('accounts'), '13'),
        tid: Ref(Collection('accounts'), '19'),
        bid: Ref(Collection('accounts'), '17'),
        delta: 500
        // transaction time is recorded automatically 
      }}
    )
  )
)
```

This FQL transaction performs the same number of logical operations as the SQL version, but in Fauna, it runs at the stronger serializable isolation level.

On a provisioned Aurora PostgreSQL primary process with 32 ACUs, [various](https://blog.dbi-services.com/aurora-serverless-v2-cpu/) third-party benchmarks suggest that Aurora can do 3,000 to 3,500 transactions per second, leading to a best-case cost for Aurora Serverless v2 of $0.30 per million transactions.

### IO costs

Aurora ACUs only represent the cost of the compute tier. Although our transaction is not actually doing any meaningful compute, its overhead, especially of the concurrency control protocol, still consumes ACUs.

Read and write IOs in Aurora are charged separately and represent the resource consumption of the storage layer. Aurora reads and writes cost $0.20 per million IOs, per region.

The benchmark above kept its working set entirely in memory, which effectively eliminates read costs from the transaction. If we assume that reads actually reach the disk, and add write IOs as well, our Aurora transaction is now costing us an additional $1.60 per million transactions for our 8 IOs.

Finally, if we are reading from the storage layer, instead of the cache, this will slow the query layer down by an order of magnitude as well, which means we need to pay for more compute time. This leads us to a real-world worst-case cost of up to $3.00 + $1.60, or $4.60 per million transactions. Workloads with good cache hit rates will be cheaper, so the typical cost is somewhere in the middle.

### Multi-region replication costs

However, that Aurora price is also only for single-region replication. Fauna replicates to three geographically distributed regions by default. If we add multi-region costs to Aurora, we need to pay an additional $0.80 per region per million transactions, for a total cost of $6.80 per million.

If we want to keep our global Aurora regions fully available, we also need to run at least one secondary read process in each of them at all times. At the previous minimum runtime costs, this will add a fixed cost of an additional $700.

Ultimately this leads us to an all-in cost of $8.50 per million transactions.

*NOTE: multi-region replication is not yet available in Aurora Serverless v2 preview.*

### Storage costs

In both Aurora and Fauna, storage is billed by stored byptes per month. Aurora charges $0.10 per GB-month per region. Fauna charges $0.25 per GB-month, which includes multi-region replication. Aurora’s equivalent would be $0.30 per GB-month. Fauna and Aurora are essentially the same in this regard, and notably, both cheaper than DynamoDB.

However, Aurora charges separately for the backtrack capability. The storage costs for Aurora backtrack are the same as for regular data. However, there is an additional write cost of $0.012 per million, slightly increasing the cost of our transaction if we want to retain rollback capability.

### Bandwidth costs

AWS charges normal bandwidth rates, specifically, $0.02 per GB to other AWS regions, and $0.09 per GB to the public internet, exclusive of volume discounts.

Additional costs for VPCs also apply.

### Miscellaneous other costs

The previous Aurora costs are exclusive of indexing, highly contended transactions, and other resource consumption that may be common in a more complex data model. Additional Aurora write amplification or consolidation can come from the write block size limit (4k) as well as background processes like compaction and monitoring.

Because the storage engine is different, it is not possible to predict Aurora IO usage based on existing MySQL or PostgreSQL installations.

And similar to DynamoDB, costs are accrued in upstream systems like AWS Lambda and other services required to run any compute components of the transaction that SQL cannot do, and to secure the database such that it can be accessed from the public internet.

Finally, the process-based model also means that costs are not linear with throughput. Smaller provisioned capacities may be less efficient per-query for equivalent datasets, due to poorer cache hit rates and fixed-cost process overhead.

## Conclusion

What then is the Aurora experience really like?

In Aurora, standard operations like upgrading the underlying database version are still user-managed concerns, and induce unpredictable amounts of write unavailability. Primary process failover takes time as well. Noisy neighbor problems vary based on process locality with other queries, not simply on logical data contention. Worrying about [threads, buffer pool hit rate, cold start time, and connection management](https://aws.amazon.com/blogs/database/planning-and-optimizing-amazon-aurora-with-mysql-compatibility-for-consolidated-workloads/) sounds uncomfortably familiar to the managed cloud experience. And finally, the application itself must still manage its own consistency concerns and ensure it is talking to primary or secondary processes at the right times.

However, if you are porting legacy RDBMS workloads and require wire protocol compatibility with either MySQL or PostgreSQL, Aurora is a direct fit. The rapid auto-scaling and reliable asynchronous replication capabilities are impressive, especially given the constraints.

Overall, there’s one thing you may have noticed about Aurora: there’s awful lot of discussion about servers for a serverless database! The provisioned nature of the legacy technology still bleeds through. Personally, I believe that Fauna’s unique architecture and features serve serverless architectures better.

Like the previous post on DynamoDB, I hope this article has provided you a clearer understanding of the motivations, architectures, and value of both Aurora Serverless V2 and Fauna. And further, that this understanding helps you make more informed decisions about which tool is right for which job.