---
layout: post
title: "Serializability vs “Strict” Serializability: The Dirty Secret of Database Isolation Levels"
date: 2019-02-21
description: "For many years “serializability” was referred to as the “[gold standard](https://amplab.cs.berkeley.edu/wp-content/uploads/2013/10/hat-vldb2014.pdf)” of database isolation levels. It was the highest isolation level offered in the vast majority of commercial database systems (so"
original_url: "https://fauna.com/blog/serializability-vs-strict-serializability-the-dirty-secret-of-database-isolation-levels"
author: "Daniel Abadi, Matt Freels"
category: blog
---

For many years “serializability” was referred to as the “gold standard” of database isolation levels. It was the highest isolation level offered in the vast majority of commercial database systems (some highly widely-deployed systems could [not even offer](https://blog.dbi-services.com/oracle-serializable-is-not-serializable/) an isolation level as high as serializability). In this post, we show how serializability is not and never was a “gold standard” for database systems. In fact, strict serializability has always been the gold standard. As we will see, because the implementation of serializability in legacy database systems has provided key strict guarantees, the difference between "serializability" and "strict serializability" has been mostly ignored. However, in the modern world of cloud-centric distributed systems, the difference in guarantees is significant.

This post first explains the concept of isolation in database systems and then illustrates the important differences between these two isolation levels, and correctness bugs introduced by “only” serializable systems that application programmers need to be aware of.

# Background: What is “Isolation”

Assume we have a ticket sales application allows people to purchase tickets to a limited supply event. The application sends transactions of the following form to the data store (when a customer attempts to purchase a ticket):

```
 IF inventory > 0 THEN
     Inventory = inventory -1;
     Process(order)
     RETURN SUCCESS
ELSE
     RETURN FAIL
```

If the data store supports ACID transactions, the above code will be processed atomically --- either the entire order will be processed and the inventory decremented, or neither. But it is impossible for anything else to happen.

If there is only one transaction running in the system, it is clear that there will not be any problems running the above code. If there is enough inventory, the order will be processed. Otherwise, it will not.

Problems typically only occur under concurrency: multiple customers attempt to purchase a ticket at the same time. If there are more customers attempting to buy tickets than there is inventory, the ideal final outcome is that all of the inventory will be sold, and no more. Indeed, overselling inventory for limited supply events make even cause legal liability for the application developers.

If the above code is allowed to run without system-level guarantees, there is a clear “race condition” in our code. Two parallel threads may run the first line at the same time and both see that inventory is “1”. Both threads thus pass the IF condition and process the order, which results in the negative outcome of an oversale of the inventory.

The system level guarantees that prevent these negative outcomes in the face of concurrent requests fall under the I of ACID --- “Isolation”.

The gold standard in isolation is “serializability”. A system that guarantees serializability is able to process transactions concurrently, but guarantees that the final result is equivalent to what would have happened if each transaction was processed individually, one after other (as if there were no concurrency). This is an incredibly powerful guarantee that has withstood the test of time (5 decades) in terms of enabling robust and bug-free applications to be built on top of it.

What makes serializable isolation so powerful is that the application developer doesn’t have to reason about concurrency at all. The developer just has to focus on the correctness of individual transactions in isolation. As long as each individual transaction cannot violate the semantics of an application, the developer is ensured that running many of them concurrently will also not violate the semantics of the application. In our example, we just have to ensure the correctness of the 6 lines of code we showed above. As long as they are correct when processed over any starting state of the database, the application will remain correct in the face of concurrency.

# The Dirty Secret of Serializability: It’s Not as "Safe" as it Seems

Despite the incredible power of serializability, there do exist stronger guarantees, and databases that guarantee “only” serializability are actually prone to other types of bugs that are only tangentially related to concurrency.

The first limitation of serializability is that it does not constrain how the equivalent serial order of transactions is picked. Given a set of concurrently executing transactions, the system guarantees that they will be processed equivalently to a serial order, but it doesn’t guarantee any particular serial order. As a result, two replicas that are given an identical set of concurrent transactions to process may end up in very different final states, because, they chose to process the transactions in different equivalent serial orders. Therefore, replication of databases that guarantee “only” serializability cannot occur by simply replicating the input and having each replica process the input simultaneously. Instead, one replica must process the workload first, and a detailed set of state changes generated from that initial processing is replicated (thereby increasing the amount of data that must be sent over the network, and causing other replicas to lag behind the primary).

The second (but related) limitation of serializability is that the serial order that a serializable system picks to be equivalent to doesn’t have to be at all related to the order of transactions that were submitted to the system. A transaction Y that was submitted after transaction X may still be processed in an equivalent serial order with Y before X. This is true even if Y was submitted many weeks after X completed --- it is still theoretically possible for a serializable system to bring Y back in time, and process it over a state of the system before X existed (as long as no committed transaction read the same data that Y writes). Technically speaking, every read-only transaction could return the empty-set (which was the initial state of the database) and not be in violation of the serializability guarantee --- this is legal since the system can bring read-only transactions “back in time” and get a slot in the serial order before any transactions that write to the database.

It is unlikely any user would use a database that always returned the empty-set for every read-only query. So how did serializability become the “gold standard”?

In the old days, databases would only run on a single machine. It was therefore trivial for database systems to make stronger correctness guarantees than serializability. In fact, it was so trivial that vendors never bothered advertising this stronger guarantee. Nonetheless, this stronger guarantee has a name: [Strict Serializability](http://cs.brown.edu/~mph/HerlihyW90/p463-herlihy.pdf).

# Strict Serializability is Much Safer

Strict serializability adds a simple extra constraint on top of serializability. If transaction Y starts after transaction X completes (note that this means that X and Y are, by definition, not concurrent), then a system that guarantees strict serializability guarantees both (1) final state is equivalent to processing transactions in a serial order and (2) X must be before Y in that serial order. Therefore, if I were to submit a transaction to a system that guarantees strict serializability, I know that any reads of that transaction will reflect the state of the database resulting from committed transactions (at least) up to the point in time when my transaction was submitted. The transaction may not see writes from concurrently submitted transactions, but at least it will see all the writes that completed before it began.

When the database sits on a single machine, guaranteeing strict serializability usually involves no increased effort relative to guaranteeing serializability. Before committing a transaction, every existing system (the one exception being Dr. Daniel Abadi’s [“lazy transactions” paper](http://www.cs.umd.edu/~abadi/papers/lazy-xacts.pdf) from 2014) must perform the writes involved in that transaction. For a transaction that comes along later, it is usually more work to ignore these writes (and not see them) than it is to see them. Therefore, just about every serializable system running on a single machine also guaranteed strict serializability. This resulted in the state of the industry where documentation of database systems would indicate that they guarantee “serializability” but in reality, they were guaranteeing “strict serializability”.

The “strict serializability” guarantee eliminates the possibility of the system returning stale/empty data that we discussed in the previous section. [It does not, however, eliminate the problem of replica divergence that we discussed in that section. We will come back to that issue in a future post.]

To summarize thus far: in reality, “strict serializability” is the gold standard isolation level in database systems. However, as long as most databases were running on a single machine, there was no perceivable difference in systems that guaranteed “serializability” and systems that guaranteed “strict serializability”. Therefore, colloquially, “serializability” became known as the gold standard and the semantic inaccuracy of this colloquialism never mattered.

# Distributed Systems Make Serializability Dangerous

In a distributed system, the jump from serializability to strict serializability is no longer trivial. Take the simple example of our ticket sales application. Let’s say that the ticket inventory is **replicated** across two machines --- A and B --- both of which are allowed to process transactions. A customer issues a transaction to machine A that decrements the inventory. After this transaction completes, a different transaction is issued to machine B that reads the current inventory. On a single machine, the second transaction would certainly read the write of the first transaction. But now that they are running on separate machines, if the system does not guarantee **consistent reads** across machines, it is very much possible that the second transaction would return a stale value (e.g. this could happen if replication was asynchronous and the write from the first transaction has not yet been replicated from A to B). This stale read would not be a violation of serializability. It is equivalent to a serial order of the second transaction prior to the first. But since the second transaction was submitted after the first one completed, it is certainly a violation of strict serializability.

Therefore, when data is replicated across machines, there is a huge and practical difference between serializability and strict serializability. It is very important for the end user to be aware of this difference before selecting a database system to use. Distributed systems that fail to guarantee strict serializability are prone to many different types of bugs.

The difference between serializability and strict serializability is much more than the “stale read” bug from my previous example. Let’s look at another bug that may emerge, called a “causal reverse”.

A bank gives you “free checking” if the total amount of money you have deposited with them (across all your accounts) exceeds $5000. Alice has exactly $5000 and needs to write a check to her child’s babysitter, so she transfers some additional money into her savings account. After getting the assurance that the transfer was successful and the money is in her savings account, she writes the check from her checking account. When she gets her statement, she sees the penalty for not keeping her balance across all accounts above $5000.

So what happened? A violation of strict serializability. The two transactions --- the addition to her savings account and subtraction from her checking account got reordered so that the checking account subtraction happened before the savings account addition, even though the subtraction was submitted after the addition completed. Such reordering is totally possible in a system that only guarantees serializability, but impossible in a system that guarantees strict serializability. Transaction reordering of this kind is very common in this type of example where the transactions access disjoint data (checking balance vs savings balance) if the disjoint data are located on different machines in a distributed system. Once the transactions got reordered, Alice’s balance across both accounts temporarily dipped below $5000, which caused the penalty to be assessed. If the bank had built their app on top of a system that guarantees strict serializability, they would not have to deal with the angry phone call from Alice.

# Fauna: Strict Serializability for All Transactions

There are very few distributed database systems that claim to guarantee an isolation level of strict serializability. Even fewer can actually achieve this guarantee with no small print. Fauna is one of those systems.

Fauna now supports the configuration to make all transactions --- both read-write and read-only transactions --- run with strictly serializable isolation.

Fauna’s guarantee of strict serializability is also the most robust on the market. Since the definition of strict serializability is based on real-time (as we discussed above: transactions submitted after the completion of other transactions in real time must respect their respective ordering), other vendors use local clocks on the different machines in the distributed system in order to enforce the guarantee. Unfortunately, approaches based on this method are only as good as the time synchronization protocol across machines and are prone to corner case violations when the synchronization algorithm temporarily exceeds assumptions on maximum clock skew. Much ink has been spilled how Fauna is able to guarantee consistency without clock synchronization, and why this approach is [better than the approach taken by Spanner and Spanner derivative systems](https://dbmsmusings.blogspot.com/2018/09/newsql-database-systems-are-failing-to.html). Fauna’s approach to achieving strict serializability also spans to multi-region environments and is similar to how it achieves consistency, through its scalable, distributed unified log. We will go into deeper technical detail in a future post.

The bottom line is that if Fauna is configured to be strictly serializable for all transactions, then stale reads will never happen. Every read issued by an application is guaranteed to reflect the writes of transactions that have completed prior to when the read was issued. This is true, no matter how much skew exists across the local clocks of the machines in the Fauna deployment. Even if one machine thinks that the year is currently 1492, any reads served by that machine will reflect all committed writes. Furthermore, transactions in Fauna never get reordered ahead of completed transactions --- even if they touch completely disjoint sets of data that are located on different machines. Therefore, the bank transaction reordering problem we saw earlier is guaranteed to never occur in Fauna.

Fauna’s support for strict serializability thus makes it the safest distributed (scalable) database system on that market.