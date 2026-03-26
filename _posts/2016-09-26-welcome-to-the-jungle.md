---
layout: post
title: "Welcome to the Jungle"
date: 2016-09-26
description: "In early 2008, I joined Twitter as a backend engineer. First order of business: adding a server timeout. Second, something to display after that timeout: the famous fail whale page. At the time, MySQL"
original_url: "https://fauna.com/blog/welcome-to-the-jungle"
author: "Evan Weaver"
category: blog
---

In early 2008, I joined Twitter as a backend engineer. First order of business: adding a server timeout. Second, something to display after that timeout: the famous fail whale page. At the time, MySQL scalability was our most critical bottleneck. Unable to find any off-the-shelf databases that would meet our needs, we did it ourselves and created the distributed data services that made tweeting reliable: the social graph, the timeline service, the image repository, the users database, and the tweet database.

These databases shared an Achilles heel. Due to extreme hardware and time constraints, each system, though incredibly efficient, was dedicated to doing only one thing–serving millions of very specific requests per second. The rocket only went one way, limiting the product and the company from adapting to new business opportunities.

> We were unable find any off-the-shelf databases that would meet our needs at Twitter.

Software infrastructure often inhibits innovation. Business people propose product improvements, but are blocked by the cost of modifying applications deployed at scale. Operators struggle to maintain reliability and efficiency as workloads change. Developers take on increasing amounts of technical debt to deliver features on time.

When we left Twitter, we were surprised and disappointed to find that this fundamental computer science problem remained unsolved. So we set out to fix it–with the same team, the same experience, and the same proven ability to deliver. Our goal was to create a new type of data system: the adaptive database. Any size, any workload, any operating environment.

> Our goal was to create a new type of data system: the adaptive database. Any size, any workload, any operating environment.

We have spent four years methodically building this solution. Although we have more work ahead of us, Fauna is production-ready and available now in managed beta in the cloud, on-premises, and in hybrid deployments. Although we are just emerging from stealth, Fauna is battle-hardened, running in dozens of data centers around the world, serving tens of millions of end-users in business-critical transactions.

For the technically-minded, Fauna is a transactional, temporal, geographically distributed, strongly consistent, secure, multi-tenant, QoS-managed operational database. It’s implemented on the JVM for portability, and it’s relational, but not SQL. Instead, it’s queried via type-safe embedded DSLs, like LINQ. We have put a lot thought into Fauna and can’t wait to explain the reasoning behind each of our design decisions to the community.

> Teams, products, and applications should coexist in one data infrastructure–just like a rainforest.

We named the company and the product Fauna because it reflects our vision of a dynamic, unified ecosystem. Teams, products, and applications should coexist, unconstrained by siloed and expensive legacy technology–just like a rainforest, where the environment supports a diverse, ever-evolving array of species, no resource is wasted, and constant adaptation is the key to success.

To this end, I am thrilled to announce our [seed round](http://www.prnewswire.com/news-releases/fauna-inc-raises-45m-in-funding-to-bring-industrys-first-adaptive-operational-database-to-market-300330693.html) and our emergence from stealth. We’re at the beginning of a great migration in data. The journey will be long, but exciting. We hope you will join us on our trek.