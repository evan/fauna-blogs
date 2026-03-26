---
layout: post
title: "Introducing Types and Enforcement to Fauna Schema "
date: 2024-04-16
description: "Fauna's types and enforcement features enable development teams to define and enforce schema structure directly within the database, merging the flexibility of document models with the strict data int"
original_url: "https://fauna.com/blog/introducing-types-and-enforcement-to-fauna-schema"
author: "Matt Freels"
category: blog
---

We are excited to announce the addition of document types to Fauna Schema (**GA as of October, 2024**). Document types enable development teams to define and enforce schema structure directly within the database - blending the flexibility of document models with the strict data integrity controls of relational databases. This rounds out our robust support for types and enforcement, which includes document types and the previously released computed fields and check constraints.

Resources

### Fauna Schema

Fauna Schema is a comprehensive suite of tools that enables development teams to define, manage, progressively type, enforce, and evolve their database schema to meet changing business needs. Explore these resources to help you get you started building with Fauna Schema.


## Completing Fauna's type and enforcement capabilities

Fauna Schema is comprised of three core elements:

* Fauna Schema Language
* Types and enforcement (including document types, computed fields, and check constraints)
* Zero-downtime migrations

Late last year we introduced Fauna Schema Language (FSL), which allows developers to define domain models, access controls, and user-defined functions in a human-readable language that is managed as a set of files in a code repository. We followed that with support for computed fields and check constraints. Computed Fields allow values of document fields to be dynamically generated based on expressions defined by the developer, and Check Constraints ensure that documents when written to the database, adhere to specific validation rules. Computed fields and check constraints are now complemented by document types, unlocking a full type system and comprehensive schema enforcement. Zero-downtime migrations, meanwhile, allows development teams to implement schema changes without service interruptions, facilitating continuous deployment and integration practices.

Let's dive into types and enforcement in more detail.

## Enforcing schema with document types

With introduction of document types, Fauna allows specifying a set of fields and types for documents in a collection as part of the collection's definition. Fauna then enforces that all documents within the collection conform to this defined shape.

You define a document type in a collection schema using field definitions and wildcard constraints.

* **Field definitions:** Field definitions represent predefined fields for a collection’s documents. These definitions consist of the field name, accepted data types, and default values. By incorporating field definitions into the collection schema’s document type, you ensure that each document conforms to a predefined structure.
* **Wildcard Constraint:** A wildcard constraint allows ad hoc fields within documents and controls accepted data types for these fields. This flexibility enables developers to introduce new field definitions and degrees of enforcement dynamically. Users can define where in the spectrum of permissive to strict they need their schema to be.

We have designed document types to naturally extend Fauna's type system to database schema: Similar to TypeScript, Fauna is *gradually typed*, meaning that it combines both static typing and dynamic typing into one system. As applied to document types, this means that Fauna blends the two extremes for data—fully structured (or static) and semi-structured (or dynamic)—into one coherent system.

```
collection Product {
  name: String
  price: Number
  category: "Tools" | "Apparel" | "Accessories"
  desc: String

  index byCategory {
    terms: [.category]
  }
}
```

Fauna's answer to schema in the form of document types is flexible and adaptable to a wide array of application data models where previously there was a choice: SQL databases, while having powerful schema capabilities, require it at all times even when it is a hindrance. For certain development styles, this is an area where document databases have an advantage, where their flexible schemaless nature is far less restrictive when iterating on a new application or feature. Conversely, once a project has started with a document database, the benefits of a relational database's schema capabilities are unavailable.

Fauna confers the advantages of both. Development can start with schemaless collections in support of iterating quickly, and gradually adopt more constraints in the form of precise document types as an application matures. Or, for applications which need to work with structured semi-structured data, Fauna can be used for both, eliminating the need for secondary systems.

In the end this leads to increased development speed as well as reduced infrastructure complexity and costs.

## Conclusion

Before Fauna, developers had to choose between relational and document databases, each with their pros and cons:

While document databases accelerate early development with a minimal, JSON-based, schemaless design, their ad hoc approach to schema enforcement and consistency results in hidden complexity and data correctness issues down the line.

Where relational databases start from a principled foundation, their rigid requirements for schema that force up-front design choices as well as architectural limitations leading to locked-in, sub-optimal schema designs hamper the development of new apps and features, and make change more costly as applications scale.

Fauna's document-relational model provides the best aspects of both approaches. Overall, Fauna provides a rich set of capabilities—including its TypeScript-inspired query language FQL and its consistency and multi-region resiliency provided by its Distributed Transaction Engine—which work together to maximize developer productivity while maintaining a high degree of application correctness. Fauna's comprehensive types and enforcement system reinforces this stack by offering the schema flexibility of document models and the strict data integrity controls of relational databases.

The end result lets developers implement higher quality applications with sophisticated feature sets, brought to market more quickly, with best-in-class operational reliability. As always, we're excited to see what you build with Fauna!