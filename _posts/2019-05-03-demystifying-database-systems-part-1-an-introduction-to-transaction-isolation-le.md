---
layout: post
title: "Demystifying Database Systems, Part 1: An Introduction to Transaction Isolation Levels"
date: 2019-05-03
description: "For decades, database systems have given their users multiple isolation levels to choose from, ranging from some flavor of “serializability” at the high end down to “read committed” or “read uncommitt"
original_url: "https://fauna.com/blog/introduction-to-transaction-isolation-levels"
author: "Daniel J. Abadi, Evan Weaver"
category: blog
---

For decades, database systems have given their users multiple isolation levels to choose from, ranging from some flavor of “serializability” at the high end down to “read committed” or “read uncommitted” at the low end. These different isolation levels expose an application to markedly different types of concurrency bugs. Nonetheless, many database users stick with the default isolation level of whatever database system they are using, and do not bother to consider which isolation level is optimal for their application. This practice is dangerous—the vast majority of widely-used database systems—including Oracle, IBM DB2, Microsoft SQL Server, SAP HANA, MySQL, and PostgreSQL—do not guarantee any flavor of serializability by default. As we will detail below, isolation levels weaker than serializability can lead to concurrency bugs in an application and negative user experiences. It is very important that a database user is aware of the isolation level guaranteed by the database system, and what concurrency bugs may emerge as a result.

> Many database users stick with the default isolation level...and do not bother to consider which isolation level is optimal for their application

In this post we give a tutorial on database isolation levels, the advantages that come with lower isolation levels, and the types of concurrency bugs that these lower levels may allow. Our main focus in this post is the difference between serializable isolation levels and lower levels that expose an application to specific types of concurrency anomalies. There are also important differences within the category for serializable isolation (e.g., “strict serializability” makes a different set of guarantees than “one-copy serializability”).

However, in keeping with the title of this post as an “introduction” to database isolation levels, we will ignore the subtle differences within the class of serializable isolation, and focus on the commonality of all elements within this class—referring to this commonality as “serializable”. In a future, less introductory, post, we will investigate the category of serializable isolation in more detail.

### What is an “Isolation Level”?

**Database isolation** refers to the ability of a database to allow a transaction to execute as if there are no other concurrently running transactions (even though in reality there can be a large number of concurrently running transactions). The overarching goal is to prevent reads and writes of temporary, aborted, or otherwise incorrect data written by concurrent transactions.

There is such a thing as perfect isolation (we will define this below). Unfortunately, perfection usually comes at a performance cost—in terms of transaction latency (how long before a transaction completes) or throughput (how many transactions per second can the system complete). Depending on how a particular system is architected, perfect isolation becomes easier or harder to achieve. In poorly designed systems, achieving perfection comes with a prohibitive performance cost, and users of such systems will be pushed to accept guarantees significantly short of perfection. However, even in well-designed systems, there is often a non-trivial performance benefit achieved by accepting guarantees short of perfection. Therefore, **isolation levels** came into existence: they provide the user of a system the ability to trade off isolation guarantees for improved performance.

> The overarching goal is to prevent reads and writes of temporary, aborted, or otherwise incorrect data written by concurrent transactions

As we will see as we proceed in this discussion, there are many subtleties that lead to confusion when discussing isolation levels. This confusion is exacerbated by the existence of a [SQL standard](http://jtc1sc32.org/doc/N2301-2350/32N2311T-text_for_ballot-CD_9075-2.pdf) that **fails to accurately define database isolation levels** and database vendors that attach liberal and non-standard semantics to particular named isolation levels. Nonetheless, a database user is obligated to do due diligence on the different options provided by a particular system, and choose the right level for their application.

### Perfect Isolation

Let’s begin our discussion of database isolation levels by providing a notion of what “perfect” isolation is. We defined **isolation** above as the ability of a database system to allow a transaction to execute as if there are no other concurrently running transactions (even though in reality that can be a large number of concurrently running transactions). What does it mean to be perfect in this regard? At first blush, it may appear that perfection is impossible. If two transactions both read and write the same data item, it is critical that they impact each other. If they ignore each other, then whichever transaction completes the write last could clobber the first transaction, resulting in the same final state as if it never ran.

The database system was one of the first scalable concurrent systems, and has served as an archetype for many other concurrent systems developed subsequently. The database system community—many decades ago—developed an incredibly powerful (and yet perhaps underappreciated) mechanism for dealing with the complexity of implementing concurrent programs.

The idea is as follows: human beings are fundamentally bad at reasoning about concurrency. It’s hard enough to write a bug-free non-concurrent program. But once you add concurrency, there are a near-infinite array of race conditions that could occur—if one thread reaches line 17 of a program before another thread reaches line 5, but after it reaches line 3, a problem could occur that would not exist under any other concurrent execution of the program. It is nearly impossible to consider all the different ways that program execution in different threads could overlap with each other, and how different types of overlap can lead to different final states.

Instead, database systems provide a beautiful abstraction to the application developer—a “transaction”. A transaction may contain arbitrary code, but it is fundamentally single-threaded.

An application developer only needs to focus on the code within a transaction—to ensure it is correct when there are no other concurrent processes running in the system. Given any starting state of the database, the code must not violate the semantics of the application. Ensuring correctness of code is non-trivial, but it is much easier to ensure the correctness of code when it is running by itself, than ensuring the correctness of code when it is running alongside other code that may attempt to read or write shared data.

> A transaction may contain arbitrary code, but it is fundamentally single-threaded

If the application developer is able to ensure the correctness of their code when no other concurrent processes are running, a system that guarantees **perfect** isolation will ensure that the code remains correct even when there is other code running concurrently in the system that may read or write the same data. In other words, a database system enables a user to write code without concern for the complexity of potential concurrency, and yet still process that code concurrently without introducing new bugs or violations of application semantics.

Implementing this level of perfection sounds difficult, but it’s actually fairly straightforward to achieve. We have already assumed that the code is correct when run without concurrency over **any starting state**. Therefore, if transaction code is run serially — one after the other—then the final state will also be correct. Thus, in order to achieve perfect isolation, all the system has to do is to ensure that when transactions are running concurrently, the final state is equivalent to a state of the system that would exist if they were run serially. There are several ways to achieve this—such as via locking, validation, or multi-versioning—that are out of scope for this article. The key point for our purposes is that we are defining “perfect isolation” as the ability of a system to run transactions in parallel, but in a way that is equivalent to as if they were running one after the other. In the SQL standard, this perfect isolation level is called **serializability**.

Isolation levels in distributed systems get more complicated. Many distributed systems implement variations of the serializable isolation level, such as **one copy-serializability (1SR),** **strict serializability (strict 1SR)** or **update serializability (US)**. Of those, “strict serializability” is the most perfect of those serializable options. However, as we mentioned above, in order to focus the discussion in this post on core concepts behind database isolation, we will defer discussion of these more advanced concepts to a future post.

### Anomalies in Concurrent Systems

The SQL standard defines several isolation levels below serializability. Furthermore, there are other isolation levels commonly found in commercial database systems—most notably snapshot isolation—which are not included in the SQL standard. Before we discuss these different levels of isolation, let’s discuss some well-known application bugs/anomalies that can occur at isolation levels below serializability. We will describe these bugs using a retail example.

Let’s assume that whenever a customer purchases a widget, the following “Purchase” transaction is run:

1. Read old inventory
2. Write new inventory which is one less than what was read in step 1
3. Insert new order corresponding to this purchase into the orders table

If such Purchase transactions run serially, then all initial inventory will always be accounted for. If we started with 42 widgets, then at all times, the sum of all inventory remaining plus orders in the order table will be 42.

But what if such transactions run concurrently at isolation levels below serializability?

For example, let’s say that two transactions running concurrently read the same initial inventory (42), and then both attempt to write out the new inventory of one less than the value that they read (41) in addition to the new order. In such a case, the final state is an inventory of 41, yet there are two new orders in the orders table (for a total of 43 widgets accounted for). We created a widget out of nothing! Clearly, this is a bug. It is known as the **lost-update anomaly**.

As another example, let’s say these same two transactions are running concurrently but this time the second transaction starts in between steps (2) and (3) of the first one. In this case, the second transaction reads the value of inventory after it has been decremented - i.e. it reads the value of 41 and decrements it to 40, and writes out the order. In the meantime, the first transaction aborted when writing out the order (e.g. because of a credit card decline). In such a case, during the abort process, the first transaction reverts back to the state of the database before it began (when inventory was 42). Therefore the final state is an inventory of 42, and one order written out (from the second transaction). Again, we created a widget out of nothing! This is known as the **dirty-write anomaly** (because the second transaction overwrote the value of the first transaction’s write before it decided whether it would commit or abort).

As a third example, let’s say a separate transaction performs a read of both the inventory and the orders table, in order to make an accounting of all widgets that ever existed. If it runs between steps (2) and (3) of a Purchase transaction, it will see a temporary state of the database in which the widget has disappeared from the inventory, but has not yet appeared as an order. It will appear that a widget has been lost—another bug in our application. This is known as the **dirty-read anomaly**, since the accounting transaction was allowed to read the temporary (incomplete) state of the purchase transaction.

As a fourth example, let’s say that a separate transaction checks the inventory and acquires some more widgets if there are fewer than 10 widgets left:

1. `IF (READ(Inventory) = (10 OR 11 OR 12))`
   Ship some new widgets to restock inventory via standard shipping

2. `IF (READ(Inventory) < 10)`
   Ship some new widgets to restock inventory via express shipping

Note that this transaction reads the inventory twice. If the Purchase transaction runs in between step (1) and (3) of this transaction, then a different value of inventory will be read each time. If the initial inventory before the Purchase transaction ran was 10, this would lead to the same restock request to be made twice—once with standard shipping and once with express shipping. This is called the **non-repeatable read anomaly.**

As a fifth example, imagine a transaction that scans the orders table in order to calculate the maximum price of an order and then scans it again to find the average order price. In between these two scans, an extremely expensive order gets inserted that skews the average so much that it becomes higher than the maximum price found in the previous scan. This transaction returns an average price that is larger than the maximum price—a clear impossibility and a bug that would never happen in a serializable system. This bug is slightly different than the non-repeatable read anomaly since every value that the transaction read stayed the same between the two scans—the source of the bug is that **additional records were inserted** in between these two scans. This is called the **phantom read anomaly**.

As a final example, assume that the application allows the price of the widget to change based on inventory. For example, many airlines increase ticket price as the inventory for a flight decreases. Assume that the application uses a formula to place a constraint on how these two variables interrelate—e.g. 10I + P >= $500 (where I is inventory and P is price). Before allowing a purchase to succeed, the purchase transaction has to check both the inventory and price to make sure the above constraint is not violated. If the constraint will not be violated, the update of inventory by that Purchase transaction may proceed.

Similarly, a separate transaction that implements special promotional discounts may check both the inventory and price to make sure that the constraint is not violated when updating the price as part of a promotion. If it will not be violated, the price can be updated.

Now, imagine these two transactions are running at the same time—they both read the old value of I and P and independently decide that their updates of inventory and price respectively will not violate the constraint. They therefore proceed with their updates. Unfortunately, this may result in a new value of I and P that violates the constraint! If one had run before the other, the first one would have succeeded and the other would would have read the value of I and P after the first one finished and detected that their update would violate the constraint and therefore not proceed. But since they were running concurrently, they both see the old value and incorrectly decide that they can proceed with the update. This bug is called the **write skew anomaly** because it happens when two transactions read the same data but update disjoint subsets of the data that was read.

### Definitions in the ISO SQL Standard

The SQL standard defines reduced isolation levels in terms of which of these anomalies are possible. In particular, it contains the following table:

| Isolation Level | Dirty read | Non-repeatable read | Phantom Read |
|-----------------|-----------|----------------------|--------------|
| READ UNCOMMITTED | Possible | Possible | Possible |
| READ COMMITTED | Not Possible | Possible | Possible |
| REPEATABLE READ | Not Possible | Not Possible | Possible |
| SERIALIZABLE | Not Possible | Not Possible | Not Possible |

There are many, many problems which how the SQL standard defines these isolation levels. Most of these problems were [already pointed out in 1995](/assets/pdfs/tr-95-51-critique-ansi-sql.pdf), but inexplicably, revision after revision of the SQL standard have been released since that point without fixing these problems.

**The first problem** is that the standard only uses three types of anomalies to define each isolation level—dirty read, non-repeatable read, and phantom read. However, there are many types of concurrency bugs that can appear in practice—many more than just these three anomalies. In this post alone we have already described six unique types of anomalies. The SQL standard makes no mention about whether the READ UNCOMMITTED, READ COMMITTED, and REPEATABLE READ isolation levels are susceptible to the lost update anomaly, the dirty-write anomaly, or the write-skew anomaly. As a result, each commercial system is free to decide which of these other anomalies these reduced isolation levels are susceptible to—and in many cases these vulnerabilities are poorly documented, leading to confusion and unpredictable bugs for the application developer.

> The SQL standard only uses three types of anomalies to define each isolation...however, there are many types of concurrency bugs that can appear in practice

**A second, related, problem** is that using anomalies to define isolation levels only gives the end user a guarantee of what specific types of concurrency bugs are impossible. It does not give a precise definition of the potential database states that are viewable by any particular transaction. There are several improved and more precise definitions of isolation levels given in the academic literature. [Atul Adya’s PhD thesis](/assets/pdfs/adya-phd.pdf) gives a precise definition of the SQL standard isolation levels based on how reads and writes from different transactions may be interleaved. However these definitions are given from the point of view of the system. The [recent work by Natacha Crooks et. al](/assets/pdfs/crooks-seeing-is-believing.pdf) gives elegant and precise definitions from the point of view of the user.

**A third problem** is that the standard does not define, nor provide correctness constraints on one of the most popular reduced isolation levels used in practice: snapshot isolation (nor any of its many variants—[PSI](/assets/pdfs/walter-psi.pdf), [NMSI](/assets/pdfs/nmsi.pdf), [Read Atomic](/assets/pdfs/ramp-sigmod2014.pdf), etc). By failing to provide a definition of snapshot isolation, differences in concurrency vulnerabilities allowed by snapshot isolation have emerged across systems. In general, snapshot isolation performs all reads of data as of a particular snapshot of the database state which contains only committed data. This snapshot remains constant throughout the lifetime of the transaction, so all reads are guaranteed to be repeatable (in addition to being only of committed data). Furthermore, concurrent transactions that write the same data detect conflicts with each other and typically resolve this conflict via aborting one of the conflicting transactions. This prevents the lost-update anomaly. However, conflicts are only detected if conflicting transactions write an overlapping set of data. If the write sets are disjoint, these conflicts will not be detected. Therefore snapshot isolation is vulnerable to the write skew anomaly. Some implementations are also vulnerable to the phantom read anomaly.

**The fourth problem** is that the SQL standard seemingly gives two different definitions of the SERIALIZABLE isolation level. First, it defines SERIALIZABLE correctly: that the final result must be equivalent to a result that could have occurred if there were no concurrency. But then, it presents the above table, which seems to imply that as long as an isolation level does not allow dirty reads, non-repeatable reads, or phantom reads, it may be called SERIALIZABLE. Oracle, has historically leveraged this ambiguity to justify calling its implementation of snapshot isolation “SERIALIZABLE”. To be honest, I think most people who read the ISO SQL standard would come away believing the more precise definition of SERIALIZABLE given earlier in the document (which is the correct one) is the intention of the authors of the document.

Nonetheless, I guess Oracle’s lawyers have looked at it and determined that there is enough ambiguity in the document to legally justify their reliance on the other definition. (If any of my readers are aware of any real lawsuits that came from application developers who believed they were getting a SERIALIZABLE isolation level, but experienced write skew anomalies in practice, I would be curious to hear about them. Or if you are an application developer and this happened to you, I would also be curious to hear about it.)

The bottom line is this: it is nearly impossible to give clear definitions of the actual isolation levels available to application developers, because vagueness and ambiguities in the SQL standard has led to semantic differences across implementations/systems.

### What Isolation Level Should You Choose?

My advice to application programmers is the following: reduced isolation levels are dangerous. It is very hard to figure out which concurrency bugs may present themselves. If every system defined their isolation levels using the methodology of Crooks et. al. that I cited above, at least you would have a precise and formal definition of their associated guarantees. Unfortunately, the formalism of the Crooks paper is too advanced for most database users, so it is unlikely that database vendors will adopt these formalisms in their documentation any time soon. In the meantime, the definition of reduced isolation levels remain vague in practice and risky to use.

> Reduced isolation levels are dangerous...the definition of reduced isolation levels remain vague in practice and risky to use

Furthermore, even if you could know exactly which concurrency bugs are possible for a particular isolation level, writing an application in a way that these bugs will not happen in practice (or if they do, that they will not cause negative experiences for the users of the application) is also very challenging. If your database system gives you a choice, the right choice is usually to avoid lower isolation levels than serializable isolation. For the vast majority of database systems, you actually have to go and change the defaults to accomplish this!.

However, there are three caveats:

1. As I mentioned above, some systems use the word “SERIALIZABLE” to mean something weaker than true serializable isolation.  Unfortunately, this means that simply choosing the SERIALIZABLE isolation level in your database system may not be sufficient to actually ensure serializability. You need to check the documentation to ensure that it defines SERIALIZABLE in the following way: that the visible state of the database is always equivalent to a state that could have occurred if there was no concurrency. Otherwise, your application will likely be vulnerable to the write-skew anomaly.

2. As mentioned above, serializable isolation level comes with a performance cost. Depending on the quality of the system architecture, the performance cost of serializability may be large or small. In a recent research paper that I wrote with Jose Faleiro and Joe Hellerstein, we [showed](/assets/pdfs/faleiro-vldb2017.pdf) that in a well-designed system, the performance difference between SERIALIZABLE  and READ COMMITTED can be negligible…and in some cases it is possible for the SERIALIZABLE isolation level to (surprisingly) outperform the READ COMMITTED isolation level. If you find that the cost of serializable isolation in your system is prohibitive, you should probably consider using a different database system earlier than you consider settling for a reduced isolation level.

3. In distributed systems, there are important anomalies that can (and do) emerge even within the class of serializable isolation levels. For such systems, it is important to understand the subtle differences between the elements of the class of serializable isolation (strict serializability is known to be the most safe). We will shed more light into this matter in a future post.