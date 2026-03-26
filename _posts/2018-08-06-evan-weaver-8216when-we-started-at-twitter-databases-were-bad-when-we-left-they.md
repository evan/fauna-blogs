---
layout: post
title: "Evan Weaver: 'When we started at Twitter, databases were bad. When we left, they were still bad'"
date: 2018-08-06
description: "If there is one thing that is true of Twitter's growing pains, it is that it used to fail. A lot."
original_url: "https://venturebeat.com/business/evan-weaver-when-we-started-at-twitter-databases-were-bad-when-we-left-they-were-still-bad/"
author: "Stewart Rogers"
category: press
---

If there is one thing that is true of Twitter's growing pains, it is that it used to fail. A lot. The now-famous "fail whale" appeared often as the platform grew. Twitter retired the fail whale in 2013, and one of the people most responsible for the stability that allowed for its removal has now launched a new, distributed database.

*"I joined Twitter in 2008 as employee number 15 and ran the software infrastructure team there," Evan Weaver, CEO and cofounder at Fauna, told me. "Twitter was at the vanguard of mobile-first adoption trends, with truly exponential growth. The infrastructure team I ran at Twitter — many of those same engineers are at Fauna today — couldn't rely on existing databases to scale the site. Vendor database software couldn't cut it. Twitter was the first user of Cassandra outside of Facebook. We hosted the first meetup. I fixed the first build. I wrote the first tutorial."*

The growing pains at Twitter came with challenges. Difficulties that Weaver initially found hard to resolve.

*"We were hoping that these first generation NoSQL systems would evolve into a broadly capable platform that we're building at Fauna," Weaver said. "But that just didn't happen. Instead, we had to build special-purpose, highly optimized distributed datastores for all of Twitter's core operational workloads. Twitter still uses some of them today, like the social graph store and the timeline store. These systems were inflexible, though, and we always wanted to have a reusable data platform we could rely on so we could stay focused on product development instead of 'basic' problems like scaling a site to a half-billion users."*

While the fail whale has now been retired, Twitter still has issues to resolve, and that's why Weaver believes database structures need to change to handle growth at the kind of scale seen in today's social media platforms.

*"When we started at Twitter, databases were bad, When we left, they were still bad," Weaver said. "Today's enterprises want the scale and flexibility of NoSQL. But they want the strong consistency, security and reliability of relational systems. FaunaDB is that platform — solving not just scale, but all the problems adjacent to scale like security, compliance, global distribution, consistency, multi-tenancy, and more, that make it so expensive to build and operate a modern digital enterprise."*

So how does a decentralized data storage platform work to provide a better solution, and what are some of the special considerations we need to make with modern database structures, given the wild differences in volume of data, types of information stored, decentralized storage systems, and more, compared to traditional databases?

*"The biggest difference to me comes down to the size of the attack surface, so to speak," Weaver said. "Every dimension of a modern database needs to be hardened beyond legacy requirements. Scale is vastly larger than in the past. Cloud/commodity hardware is unpredictable. Networks are globally distributed. Services are exposed to uncontrolled and malicious third-parties via APIs. Even within a company, private clouds and multi-tenant architectures compete for hardware and share data across products and teams. Just like the RDBMS revolution of 40 years ago, these problems require a ground-up approach for the cloud-native 21st century."*

Those challenges require a different way of thinking and a new way of storing data in the cloud.

*"A modern database must be designed to scale and perform without compromising on capabilities offered by traditional databases, and it must operate easily in cloud environments, or hybrid, irrespective of the host," Weaver said. "A modern database need to replicate data across distributed data centers easily. It must do so without requiring a Ph.D. to deploy, operate, and manage. It must also offer developer friendliness and schema flexibility, without loss of relational data access. A modern database is a platform, not a niche data solution."*

One issue with databases in general is security. All too often, we hear of data breaches and loss of data to bad actors. According to a recent [study by IBM Security and Ponemon Institute](https://www.ibm.com/security/data-breach/), the cost of "mega breaches," where 1 million to 50 million records are lost, can run from $40 million to $350 million.

What does FaunaDB provide to ensure security and protect against data breaches?

*"Unlike other databases that default to insecure configurations or aren't possible to secure at all without additional software components or third-party tools, FaunaDB offers best-in-class security out of the box, with TLS/SSL encryption on the wire, full-disk encryption at rest, and a logical security model that guarantees that all access is limited, authenticated, identified, access-controlled, auditable, and even rewindable if you choose," Weaver said. "We are working on end-to-end per-tenant encryption to protect data even from malicious operators within a trusted environment; that will be another industry first."*

FaunaDB recently raised $25 million in series A funding, led by Point72 Ventures, to help make this vision a reality. So what's next for FaunaDB, and how is the funding being spent?

*"We are growing the company and focusing on making our customers successful," Weaver said. "Our customers span the enterprise as well as SMBs. To support growth, we're expanding the sales, customer success, and marketing teams. We continue to build out FaunaDB's capabilities to be best-in-class. We're maturing our serverless cloud offering. And we have a lot of exciting features directly applicable to cloud-native development trends coming soon. To do so, we're opening a second office on the East Coast to expand access to engineering talent. I couldn't be more excited for the next year because nobody else is doing what we are doing."*

FaunaDB is offered as a serverless cloud product spanning AWS, GCP, and soon Azure, as well as an on-premises/hybrid-cloud package for the enterprise.
