---
layout: post
title: "Introducing the future of Fauna’s database language"
date: 2023-04-25
description: "Fauna is introducing a bold new version of our database language: FQL v10. Get more done with less code and fewer queries and be able to more easily scale within or across regions without the operatio"
original_url: "https://fauna.com/blog/introducing-fql-x-database-language"
author: "Matt Freels"
category: blog
---

Today, Fauna is introducing a bold new version of our database language FQL.

Our first step on the journey to creating FQL was to internalize the feedback we have received from the 3,000+ teams that have developed and scaled their applications on Fauna over the years. We wanted to deeply understand where they were challenged with the current language and also look at how we could take the language, and their experience, to the next level.

Our goal was to deliver a modern database language that allows developers to tap into the power of Fauna’s architecture so that they can get more done with less code and scale their applications within or across regions without the operational overhead traditionally associated with a database.

We took that feedback and also carefully analyzed and drew inspiration from popular programming and query languages like TypeScript. We leveraged some of the best attributes of these languages in order to create a database language for Fauna that was immediately familiar, yet still laser focused on allowing developers to write clear and concise queries and transactional code. Developers that are familiar with TypeScript or a similar language should find FQL v10 to be very approachable.

Throughout our design and development process we iterated on the language in partnership with hundreds of Fauna and non Fauna developers. We validated the design and our implementation as we progressed in alpha and private beta programs. Today we are super excited to be moving to a broader, public beta and to start supporting a smaller number of customers in production as they go live with FQL v10 in their application.

### A language for queries and transactional business logic

FQL was designed to support relational features such as foreign keys, views, and joins and combines the ability to express declarative queries and functional business logic in transactions that are strongly consistent across geographic regions. Pushing the boundaries of distributed databases to new heights, FQL v10 is specifically designed with performance and consistency in mind, making it ideal for interactive applications, IoT services, edge computing, and other operational use cases.

As mentioned earlier, an important part of our design process for FQL v10 was to analyze popular languages such as TypeScript to understand what we could learn from them that could apply to Fauna’s domain specific database language. While each of these languages has their strengths (which we definitely wanted to borrow), they also came with their own set of tradeoffs which made them impractical to extend or adopt wholesale as a database language. In order to have a language for Fauna which was laser focused on empowering developers to write clear, concise queries and transactional code, we needed a database language that was inspired by these languages but was purpose-fit for Fauna.

The resulting FQL is familiar and easy to learn, scales to handle complex business logic, and makes it easy for developers to tap into the power of Fauna's operational database architecture.

FQL v10 features a new TypeScript-inspired syntax (extended to support concise relational queries), is optionally statically typed, has a new interface for working with collections and documents, and a features a powerful new indexing system which enables an iterative approach to developer-driven query optimization.

In order to integrate FQL v10 into applications, we have built a new set of lightweight drivers that support secure, dynamic generation of queries through programmatic composition. In addition, FQL now supports direct HTTP API access — both of which are powerful and flexible tools for integration with any application code.

## Learn once, write anywhere syntax

As part of FQL v10, we took the opportunity to rethink how we can better enable developers to more easily write queries. While developers have loved FQL's expressiveness and applicability to a wide variety of database programming tasks, the embedded DSL approach came with its challenges. Customers have frequently asked for the ability to port a query from one context to another, whether in order to interactively develop a query in the dashboard shell and copy it to their app, or be able to apply their knowledge of FQL in a new host programming environment. Furthermore, the constraints of each host language puts limits on FQL's clarity in their respective environments.

FQL v10 introduces a new dedicated syntax which applies to all contexts in which it exists. Our drivers implement a secure, string-based template system for embedded queries, meaning developers can now write their query once, and apply it everywhere.

```
Book.where(.genre == 'Fiction').order(.title)
```

Unlike SQL, which is keyword heavy and requires learning new syntax for every feature, FQL has a small core syntax — drawing from languages like TypeScript — with dedicated shorthand for common query concepts such as record predicates and projection. If you are familiar with TypeScript or a similar language, chances are you will feel right at home with FQL.

Querying, data manipulation, and other capabilities are then exposed as APIs on top of this core syntax. The end result is an easy to learn language with a rich feature set that remains discoverable over time.

> “As someone with experience in SQL and JavaScript, I find real similarities between FQL v10 and these familiar languages. FQL v10 allows for a seamless transition from writing application code to database code without requiring a context shift.”

## Streamlined query interface

One thing our customers have appreciated about FQL is that unlike SQL, query performance is apparent based on query structure. Explicit index usage and a structure like normal code make it easy to understand how a query will execute based on reading it. Combined with Fauna's strongly consistent distributed architecture, queries written in FQL are performant without having to compromise application access patterns by, for example, denormalizing or manually partitioning data. However, with customers we found that requiring the creation of indexes out of the gate could slow down initial development where specific access patterns were in flux, and the specific indexes required were not immediately apparent.

FQL v10 introduces a new core API for querying collections and collection-centric indexing system, and a streamlined, high-level interaction model for working with documents.

Collections in FQL v10 expose an ORM-like API for creating declarative queries, with composable methods for core functionality of filtering, ordering, and transforming sets of documents. This core API is enhanced with dedicated syntax to optimize the readability of predicates and projection/transformation.

```
// Select title and published date of sci-fi books, ordered by title
Book.where(.genre == 'Scifi').order(.title) { title, publishDate }
```

Working with documents themselves is as simple as reading fields off objects. While Fauna has long had support for first-class foreign keys, they still had to be explicitly read. FQL v10 simplifies them by automatically dereferencing documents through associated fields. For example if a Book has an Author stored in an 'author' field, the Author document is retrieved transparently through field access.

```
// get the name of the author of a specific book
let book = Book.firstWhere(.title == 'Left Hand of Darkness')!
book.author.name
```

All of the above — along with the fact that FQL responses are not limited to tabular results — means that it is easy to build up sophisticated join-like queries that remain easy to work with as they scale in complexity and can be tailored to return the exact data the application requires.

```
Book.where(.genre == 'Scifi').order(.title) { 
  title,
  author {
    name
  },
  publishDate
}
```

The overall result is powerful, concise, and easy to maintain code. Matt Haman, CTO of Rownd, explains,

> “With FQL v10, Fauna marries simplicity with power. Now, we can do aggregation and projection more effortlessly than in MongoDB. FQL v10 outshines SQL with minimal effort and complexity."

## Collection-oriented indexes

As mentioned above, FQL v10 has a new indexing system centered on collections. Indexes are simpler to define, simpler to use, and retain the transparency and explicit control that Fauna's indexes have provided from day one.

Structurally, FQL v10 indexes are very similar to FQL's current indexes, but have two key improvements: All FQL v10 indexes are defined on collections and are exposed as methods on the collection and every index method returns a set of documents of the collection.

For example, say we start with a query to retrieve all authors with a given last name, ordered by first name:

```
Author.where(.name.last == 'Le Guin').order(.name.first) {
  id,
  name
}
```

We can then add an index which uses last name as a lookup term, and orders Author documents for each last name by first:

```
Author.definition.update({
  indexes: {
    byLastAndFirstName: {
      terms: { field: '.name.last' },
      values: { field: '.name.first' }
    }
  }
})
```

The result is that index-based queries are then drop-in replacements for their declarative equivalents. This makes it easy to refactor code to use indexes, and makes it clear that an index is being used, as opposed to being left to the whims of a query planner:

```
Author.byLastAndFirstName('Le Guin') { 
  id,
  firstName,
  lastName
}
```

> “As a database developer, I find Fauna's use of named functions for accessing indexes brilliant. It adds flexibility and versatility to my workflow, allowing efficient interaction with indexes using custom functions,” explained Yacine Hmito, Head of Technology at Fabriq.

Yacine was excited about this part of our design and we hope you will be too.

## Improved error responses

FQL v10's error responses have been redesigned to provide much more context on where an error occurs and why. FQL v10 error responses in interactive contexts point to the exact part of a query which caused an issue. The output speaks for itself:

```
> if ('oops') Author.create({})

error: expected type: boolean, received string
at *query*:1:5
  |
1 | if ('oops') Author.create({})
  |     ^^^^^^
  |
```

We have also added logging and debug utility functions to FQL v10 in order for developers to inspect and gain insight into their queries' runtime behavior. These utilities are of course integrated with Fauna's attribute-based access control system so that developers have complete control over who has access to diagnostic information.

## Better feedback through types

FQL v10 adds support for static typing of queries and user-defined functions. Fans of statically typed languages have known for years the benefits of a good type system to make it easier to write robust code. FQL v10 is gradually typed, similar to TypeScript, meaning that certain use cases such as highly dynamic data can opt out of static typing and instead rely on runtime checks. It is also possible to enable or disable type checking entirely on a per-query or per-database basis.

Beyond better error feedback, FQL v10's type system is the basis for powerful tools like context-aware code complete which infer the types of values within a query and present in-line possible actions that can be taken on them.

Overall, as a source of better feedback and the basis for more intelligent tooling, we believe FQL v10's type system will be a boon for greater developer productivity.

## Lightweight drivers

For FQL v10, we have created a completely new lightweight driver architecture and implementations in TypeScript, Go, and Python. The resulting drivers are as low as 10-20% of the size of their current FQL equivalents, which reduces resource consumption and payload size in environments such as browsers or edge compute functions.

In addition, the underlying HTTP API is simple enough to call directly within resource-constrained application environments such as IoT or via no-code platforms. This new wire protocol and HTTP API have been extensively documented, and our drivers are liberally open-source, making it possible for us along with Fauna's community to bring support for FQL v10 to more host languages over time.

## Bringing the power of relational querying to documents

Fauna uniquely combines the power of relational querying with the flexibility of documents into a single, unified model, providing the ultimate in data storage flexibility and query efficiency for modern operational applications.

Traditionally, the definition of a relational database was tightly coupled to a very structured format of storing data in rows, columns, and tables — a format not well suited for the semi structured data we find at the core of today’s user and IoT applications. Document databases introduced the ability to work with JSON-like, semi-structured data which made them very convenient for fast initial development and intuitive for application developers. However their rigid querying capabilities and ad-hoc approach to consistency resulted in significant complexity and headache as applications scaled up and matured.

Relational databases provide great power for advanced practitioners familiar with SQL and the relational model, but the opaque performance profile, the rigid constraints of strict schema and tabular data, the dependence on ORMs to make it practical to consume non-trivial query results in application code, and — let's face it — the irregularity of SQL itself, all get in the way of the average developer's ability to take advantage of these systems beyond simple SELECT queries.

FQL and Fauna make it possible to write code which combines relational querying with the document model, creating query result shapes which transcend both. The language's composability and modularity, along with Fauna's comprehensive implementation of strong consistency, means that even as FQL code scales in terms of both size and complexity, it remains easy to understand, maintain, and extend. These attributes taken together free applications from database consistency concerns, let applications push down business logic to run close their data, and tailor query results to be exactly the data and result shape the application needs.

## Conclusion

We designed FQL v10 in response to the feedback from our customers while drawing inspiration from the best of modern programming languages in order to create a modern database language dedicated to operational workloads that also avoids the shortcomings of SQL. It brings to Fauna a new level of simplicity, expressiveness, performance, and developer productivity.

Development teams building applications on top of Fauna will get more done with less code and fewer queries and be able to more easily scale within or across regions without the operational overhead associated with managing a database.

Powered by Fauna's distributed architecture and our API delivery model, applications can achieve this without having to make tradeoffs in terms of efficiency and performance.

Visit our documentation to try out FQL v10 now in Public Beta with non-production data and get ready for a seamless transition to the future of Fauna Query Language.