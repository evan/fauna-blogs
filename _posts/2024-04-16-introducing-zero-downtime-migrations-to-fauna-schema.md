---
layout: post
title: "Introducing Zero-Downtime Migrations to Fauna Schema "
date: 2024-04-16
description: "Fauna's zero-downtime schema migrations enables seamless schema changes without service interruptions. This feature supports continuous deployment, ensuring your application evolves without downtime. "
original_url: "https://fauna.com/blog/introducing-zero-downtime-migrations-to-fauna-schema"
author: "Matt Freels"
category: blog
---

We are excited to announce the addition of zero-downtime migrations to Fauna Schema (**GA as of October, 2024**). This functionality allows development teams to implement schema changes without service interruptions, facilitating continuous deployment and integration practices. Zero-downtime migrations ensure that your applications can evolve and adapt to changing requirements without the downtime and disruption traditionally associated with schema changes.

Resources

### Fauna Schema

Fauna Schema is a comprehensive suite of tools that enables development teams to define, manage, progressively type, enforce, and evolve their database schema to meet changing business needs. Explore these helpful resources with get you started building with Fauna Schema.


## Complementing Fauna Schema with zero-downtime migrations

Fauna Schema comprises three core elements:

* Fauna Schema Language
* Types and enforcement (including document types, computed fields, and check constraints)
* Zero-downtime migrations

Late last year we introduced Fauna Schema Language (FSL), which allows developers to define domain models, access controls, and user-defined functions in a human-readable language that is managed as a set of files in a code repository. We followed that with support for computed fields and check constraints. Computed Fields allow values of document fields to be dynamically generated based on expressions defined by the developer, and Check Constraints ensure that documents when written to the database, adhere to specific validation rules. Fauna’s types and enforcement capabilities was completed with document types, released along with zero downtime migrations, which enable you to define and enforce schema structures directly within the database.

## A zero-downtime migration system

Along with document types, we built a migration system which is capable of affecting large schema migrations safely, without downtime. We considered this to be a key component in making document types—as part of Fauna's overall schema capabilities—practical to use and enhance developer productivity. To understand why, it is worth taking a look at the respective approaches usually taken by relational and document databases.

For relational databases the set of columns and types of a table are an integral part of its definition, including its physical layout on disk. In terms of modeling, the table's definition provides a guarantee that data will always conform to the shape the application expects. Therefore, when a table is altered in a traditional SQL database, existing rows must be migrated to conform to the new definition. If it cannot be (for example, adding a non-nullable column with no default), the alter command must be rejected.

Furthermore, due to architectural limitations of most mainstream relational systems, it becomes difficult or impossible to change a table's definition as your dataset grows. The alter table command will lock the table to perform the migration in most cases, incurring significant downtime at scale. Database schema becomes locked in as your application grows, resulting in reduced developer productivity and increased cost and complexity: The database no longer works for the application but against it.

Document databases, if they do support schema in some form, take a far more relaxed approach euphemistically called "schema validation", which is a form of write-time validation since document databases remain schemaless by nature under the hood. When a collection's schema definition is changed in a document database, existing documents are left untouched.

It is then up to the developer to manually migrate existing documents afterwards. The overall result is more or less the schemaless state where document databases started from, where the application itself must take on or duplicate schema management and data migrations.

With Fauna, we took the stricter approach where schema is considered to be a contract between the database and the application. Whenever a collection definition changes, Fauna will migrate every existing document in the collection to conform to the new shape. These migrations are natively zero downtime, with no risk of locking up a production database for some indeterminate amount of time. The migration is handled entirely within the database itself, avoiding the need to drag data from the DB to a client and then write it back again. From the perspective of the application, the migration is instantaneous: Physical document data is lazily migrated on demand or incrementally in the background.

Despite this strict approach, Fauna preserves development agility and reduces the risk of operational mistakes. Fauna's zero downtime migration system means that your application never has to choose between "maintenance mode"—being unavailable—or being locked in to a database schema that has long since been outgrown and is no longer a source of truth for your application or business as a whole.

### Declarative, schema-as-code migrations

We designed Fauna's schema definition language (FSL) from the ground up to support a style of database management that is better aligned with the modern software development and operational management practices we call schema-as-code. With FSL, your database schema definitions live alongside your application source code and can be managed with the same pipeline-based deployment tooling your application code is.

Fauna's migration system is integrated with FSL in order to allow you to encode explicit migration steps as part of a collection's definition. In doing so, Fauna preserves the ability to manage your application schema as code while allowing your schema to express more complex migration actions than are possible in a purely declarative approach, such as renaming a field or backfilling a new field.

```
collection Product {
  name: String
  price: Number
  category: "Tools" | "Apparel" | "Accessories"
  description: String
  quantity: Number

  index byCategory {
      terms: [.category]
  }

  migration "2024-04-15: rename desc, backfill quantity" {
      rename desc description
      backfill quantity = 0
  }
}
```

Fauna checks each new FSL submission for internal consistency as well as migration compatibility with the database's current schema, and if all checks pass, applies the schema update. In development, this workflow provides a great degree of simplicity, since schema can be uploaded multiple times without worrying about prior database state.

In critical environments such as production, Fauna's migration system plugs into Fauna's existing "protected" mode for databases which enforces a higher degree of safety. Potentially dangerous actions must be explicitly annotated in FSL code. For example, uploading a new collection definition which has fewer fields than the current one will be rejected unless the FSL includes an explicit migration statement marking the field as dropped.

Ultimately Fauna's schema-as-code approach, along with database protection, increases the safety of production environments, and allows you to leverage existing tooling such as established CI/CD pipelines to increase the security and auditability of database changes.

## Conclusion

Fauna's introduction of zero-downtime migrations marks a significant milestone in our commitment to providing robust, flexible, and enforceable schema capabilities. This feature, along with the Fauna Schema Language and types and enforcement, empowers development teams to iterate and scale their applications without the fear of service interruptions or downtime. By seamlessly integrating schema changes into continuous deployment workflows, Fauna ensures that your database evolves alongside your application, maintaining data integrity and operational efficiency.