---
layout: post
title: "Fauna’s 2024 Product Recap"
date: 2024-12-26
description: "The truly serverless database that combines the power of a relational database with the flexibility of JSON documents."
original_url: "https://fauna.com/blog/faunas-2024-product-recap"
author: "Wyatt Wenzel"
category: blog
---

Fauna took major strides this year to enhance developer productivity, streamline schema management, and simplify integrating our database into modern application stacks. From flexible schema tooling and zero-downtime migrations to new marketplace listings and ecosystem integrations, we focused on giving developers the building blocks they need to move faster and do more—all without sacrificing consistency, performance, or reliability.

### Flexible & Enforceable Schema Management

Fauna introduced a comprehensive set of schema features designed to help teams model data confidently and iterate with ease—capabilities that can’t be replicated by traditional document or relational databases without significant tradeoffs.

While older paradigms force a binary choice between rigid schemas and completely schemaless approaches, Fauna’s document-relational model provides genuine optionality. These capabilities include computed fields for dynamically generating values at query time, check constraints to enforce custom data integrity rules, and types & enforcement to bring strong schema definitions to a document-based storage model. This ultimately allows teams to start schema-less and gradually tighten their schema controls as their application evolves. All of this is anchored on zero-downtime migrations, making it possible to evolve your database schema without interrupting service availability.

Over the course of the year, these features converged into a unified schema experience, culminating in a general availability release that empowers development teams to reshape and refine their data models with unprecedented confidence and agility.

### Event Streaming and Change Data Capture

We introduced a new event streaming to help developers build more dynamic, real-time experiences. This capability ensures that applications can stay in sync with changing data and react to business events as they occur. Later, we added Change Data Capture (CDC) support, providing a persistent, queryable history of data modifications. This unlocks powerful new workflows for auditing, analytics, and triggering downstream processes – all delivered within Fauna’struly serverless model, eliminating the need to manage complex infrastructure or integrate additional services.

### Deeper Integration With the Tools You Love

To meet developers where they are, Fauna expanded its integrations and ecosystem support. We rolled out a native integration with Cloudflare Workers, making it easier to build globally distributed edge applications without sacrificing data consistency or performance. Additionally, a new Datadog integration lets teams monitor and gain insights into Fauna’s behavior alongside their broader application stack. These integrations help ensure that Fauna fits seamlessly into your existing workflows, no matter how you’ve structured your engineering environment.

### Marketplace Expansion for Flexible Deployment

Fauna made it easier to access and adopt our platform through established marketplaces, enabling teams to streamline their procurement processes. We joined the Google Cloud Marketplace, allowing GCP customers to use Fauna under their familiar billing model and consolidate usage. We also introduced a Pay-As-You-Go listing on AWS Marketplace, making it effortless for AWS users to get started with Fauna’s globally distributed, serverless database in a way that aligns with their financial and operational preferences.

### Fauna CLI v4.0

And finally, hot off the press and released just before the New Year, we also delivered a significant update to the Fauna CLI (v4.0, currently in beta). This version focuses on developer experience improvements, including simplified authentication and configuration, text highlighting, and a more intuitive workflow. The updated CLI supports managing databases and schemas with .fsl files, running queries interactively or from files, and launching local Fauna containers with ease. These enhancements ensure that teams can integrate Fauna into their daily routines more smoothly, reducing friction and accelerating their development cycles.

### Looking Ahead

Fauna’s 2024 product updates focused on reducing friction, increasing optionality, and empowering developers to build without boundaries. From refined schema capabilities and smooth migrations to real-time event streaming and robust integrations, Fauna now offers an even more powerful and flexible platform for modern application development. As we look forward, we remain committed to driving innovation that helps you deliver exceptional experiences to your users—no matter where they are in the world or how your requirements evolve.