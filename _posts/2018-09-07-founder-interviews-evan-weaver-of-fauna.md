---
layout: post
title: "Founder Interviews: Evan Weaver of Fauna"
date: 2018-09-07
description: "Previously at [Twitter](https://hackernoon.com/company/twitter) as employee #15, Evan Weaver has recently raised a $25M Series A to grow Fauna, a cloud-native database."
original_url: "https://hackernoon.com/founder-interveiws-evan-weaver-of-fauna-73f9e220ea91"
author: "Davis Baer"
category: press
---

*Previously at Twitter as employee #15, Evan Weaver has recently raised a $25M Series A to grow Fauna, a cloud-native database.*

**Davis Baer****: What’s your background, and what are you working on?**

**Evan Weaver**: My name is Evan Weaver and I’m the CEO and Co-Founder of Fauna. My passion for software development started when I was 8 or 9 years old; like many of us, I was trying to make games. I think I still have the QBasic book that I used to write a slot machine program that featured…wait for it…graphics. My father worked for a university so we had early access to the internet as well; you could download shareware from Gopher with Archie and Veronica, which were frankly terrible.

At school, I studied poetry writing, philosophy, jazz, and computer science; as you can guess, only one of these things makes money, so here we are. I got my masters in CS and worked on gene orthologs in chickens.

Along the way, I got interested in open source and Ruby on Rails and got a job at CNET Networks. I worked as a DBA and backend developer on CHOW and UrbanBaby, along with Chris Wanstrath and P. J. Hyett. You may know them from a small project called GitHub.

In 2008, I left CNET and joined Twitter as employee #15. I ended up building and managing the backend infrastructure team. We did all the distributed storage for the core business objects and worked on performance in Ruby and Hotspot.

**What motivated you to get started with your company?**

Twitter was one of the last great internet companies to be born before the cloud era. It seems crazy now, but there was no hardware available on demand. And Twitter’s rise tracked SMS and smartphone adoption curves, instead of broadband penetration like [Facebook](https://hackernoon.com/company/facebook). This meant that we had uncontrollable user growth, without natural network segmentation. We racked new hardware every 6 months, and increased end-to-end performance by almost 20% month over month, but our servers still ran at 100% utilization all the time because of latent demand especially on the API.

We couldn’t find off-the-shelf systems that could meet the scalability challenges we faced for love or money, so we were forced to build a number of point solutions. A decade later Twitter still uses some of these systems, like the social graph and the timelines database, which is processing HFT-level workloads in real time.

At Twitter, we were some of the original adopters of NoSQL as well. My team was the first user of Cassandra outside of Facebook. We fixed the build; I wrote the first tutorial; we hosted the first meetup. We were hoping that NoSQL, and Cassandra specifically, would grow into a general purpose, operationally resilient, modern cloud-native database platform — and for a variety of reasons it just didn’t happen.

Because of time and business constraints at Twitter, and the failure of the first generation of NoSQL vendors, we never got a truly reusable data platform that could meet our scalability, efficiency and global availability goals, and remain flexible for product development.

This platform is FaunaDB.

FaunaDB is a ground-up rewrite of what a modern, relational, cloud-native database should be — focused on what application developers and operators need for the future, instead of legacy constraints.

**What went into building the initial product?**

Matt Freels, my co-founder, was on my team at Twitter. We left Twitter and founded Fauna in 2012. [NVIDIA](https://hackernoon.com/company/nvidia) was our first partner; we did a lot of co-development and non-recurring engineering with them while in stealth mode to make sure we were meeting a real market need. It turns out databases need a lot of R&D so this took a little longer than we expected.

FaunaDB is built for mission-critical workloads, so proof from early adopters such as NVIDIA, CapitalOne and Nextdoor has been crucial. We’re now able to show that the FaunaDB system meets the goals we laid out for ourselves in the early days of building an enterprise-grade, relational NoSQL solution.

FaunaDB is implemented in Scala and runs on the JVM. We are happy with our choice of Scala. When we started FaunaDB, we looked at other options like C++, Go, and Rust. We wanted a modern, memory-safe language that offered a high-level type system — since databases are monoliths — but also was capable of predictable, high performance.

The JVM meets both these goals and is very friendly to deploy in any operational environment — scaling down to laptop size or up to mainframe size. It is also OS-independent, so FaunaDB is incredibly portable. Write once; run anywhere — the dream of the ’90s is alive at Fauna. Scala solves the type system problem and lets us keep the the codebase very clean and well-encapsulated.

In order to get Fauna off the ground, we used a mix of funding. We invested some of our own money and did some scalability consulting for companies that represented our target market. This was a great way for us to develop our understanding of business problems, rather than just replicate some internal system from Twitter.

We found that the market was wildly behind even the worst systems at Twitter, and it was critical to deliver a truly *general purpose* database that didn’t give up any of the flexibility of relational systems. And most companies’ database hardware is running at 10% utilization or less, which was shocking to us, and led us to incorporate FaunaDB’s multi-tenancy and QoS features for workload consolidation and safety at scale.

We got far with the mix of self-funding and partnership revenue. Once we were ready to bet the company on a specific product vision, we went out to raise venture capital so we could focus on product development full-time.

In 2016, we raised our seed round from CRV and launched FaunaDB Serverless Cloud. Just under a year ago, we raise our A round from Point72 and [Google](https://hackernoon.com/company/google) Ventures and are now in midst of our enterprise go-to-market, which is going well.

**How have you attracted users and grown your company?**

Most operational databases launch with exactly *zero* production users, and fall flat on their faces at the first production use. One of the things we did in order to make sure we really delivered on high availability, durability, and correctness, was to eat our own dogfood by operating FaunaDB Serverless Cloud.

FaunaDB Cloud is exciting because it’s a serverless, utility computing solution — again with the ’90s dreams. Our cloud is a single, multi-region, multi-provider installation of FaunaDB Enterprise, from which we dynamically provision individual tenants. There is no orchestration or container system around it and it transparently scales across AWS and Google Cloud. FaunaDB Cloud let us get direct insight into early product use cases for FaunaDB as well as harden its operations far and make it truly production-ready.

At Twitter, we operated our mission-critical systems ourselves so we are as much DBAs and operators as developers. There’s no other way to do it if you are pushing the boundaries of information science like we did with Twitter at scale or with FaunaDB at consistency and transactionality.

**What’s your business model, and how have you grown your revenue?**

Part of our vision is to package and deliver FaunaDB is every way that the market wants to get it. We value and support the smallest, newest hobbyist as well as the largest enterprise, and pricing spans that range as well.

For the individual developer we have the serverless cloud, which is pay-as-you go with no fixed cost. We just simplified our pricing model as well. And we will soon be releasing a free download for people to try out FaunaDB on-premises, or for development purposes locally.

At the same time, we offer the full-service, enterprise customer success experience for companies like Capital One who are building mission-critical systems on FaunaDB. We do everything we can to ensure that if you’re making a bet on a new operational database like FaunaDB, it pays off long-term.

One thing we’ve [learn](https://hackernoon.com/u/learn)ed over and over is that no one is happy to be in the market for a database. They are in the market out of desperation and necessity. An operational database is like the foundation of your house — you never want to change it. This means that customers want to have a real, long-term vendor partnership — a marriage, so to speak.

It’s very different from markets like analytics databases which are easy-come, easy-go because they aren’t the canonical storage for anything, or time-series databases that only store low-value data that is consumed in aggregate.

**What are your goals for the future?**

So far, we’re very happy with our company growth. We’re bringing on real, production, paying customers, including enterprise, small businesses, and individuals. We’re hiring engineers and salespeople after raising the largest series A in our space — $25M — less than a year ago.

Our immediate focus is the enterprise readiness of the company and product overall. You can expect to see a lot more from us this coming year, especially integration tooling, even more query capabilities, better enterprise and developer support, documentation and other educational materials, etc. — everything required for “turnkey” adoptability.

In the second half of the year, you will see us launch more cutting edge features like real-time streaming, locality control, and some other thing in stealth, as well as extend FaunaDB Serverless Cloud to [Microsoft](https://hackernoon.com/company/microsoft) Azure.

**What are the biggest challenges you’ve faced and obstacles you’ve overcome? If you had to start over, what would you do differently?**

Our biggest challenge, which is systemic in the startup ecosystem these days, is hiring. When I was a hiring manager at Twitter, I would routinely recruit single-income families to move to the San Francisco Bay Area. Those days are gone. We pay market rates but we are also competing with offers from publicly traded companies like [Amazon](https://hackernoon.com/company/amazon), Google, and even Twitter which issue liquid equity as RSUs. It’s hard to turn down that guaranteed payout when cost of living in the Bay Area has practically tripled.

In response, we’ve invested a lot in building a remote-first culture so we can hire the talent we need in to make Fauna a huge success no matter where people want to live, and we are opening a second office in the Boston area as well.

We are a deep tech business and we have real R&D costs, both money and time. We’re always looking for the right people, whether team members or customers, to contribute to delivering on our long-term vision and replace 40 years of obsolete, mainframe-oriented database technology.

**Have you found anything particularly helpful or advantageous?**

We’ve been the beneficiary of a lot of advice, whether the advice-giver knew it or not. We’ve watched others’ mistakes in our market very closely. Most new database entrants go to market too early. They don’t understand how long it takes to actually build a new piece of foundational technology. A distributed database is as complex as an operating system, and more error-prone.

You can raise money on a deck and a dream, but if you don’t actually have enough runway to build to that dream, you will fail. We’ve worked hard to find patient investors who understand the long-term value of what we’re building.

**What’s your advice for entrepreneurs who are just starting out?**

There’s no instant market traction in a deep tech business. Even hobbyists want their drone to stay in the air, their self-driving car to be safe, their database to be durable and consistent.

You have to do a ton of deep grinding engineering work to smooth out the rough edges on this type of project. You need to be very mindful of the potential path dependence in your implementation and architectural choices. You have to understand how long and challenging your go-to-market path will be, especially in the enterprise.

We may not be the smartest people in the room, but we are definitely the people that work the hardest and never give up. For these types of R&D intensive projects, slow and steady wins the race.

**Where can we go to learn more?**

To learn more about Fauna, check out www.fauna.com.

