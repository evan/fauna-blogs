---
layout: default
title: Fauna Blog Archive
---

_This site is an archive of the key technical writing and announcements published on the Fauna blog between 2016 and 2025._

Fauna was a globally distributed, transactional database built from scratch for modern cloud applications. The company was founded in 2012 as a consultancy by Evan Weaver and Matt Freels, engineers who had scaled Twitter's data infrastructure. Fauna set out to solve a problem they'd seen firsthand: existing databases forced painful tradeoffs between consistency, global availability, and developer experience.

The result was a document-relational database that offered strictly serializable ACID transactions across geographic replicas without relying on synchronized clocks. At the time, this was widely believed to be impossible, and other database vendors that claimed even lower levels of transaction support [were](https://jepsen.io/analyses/postgresql-12.3) [typically](https://aphyr.com/posts/294-call-me-maybe-cassandra) [lying](https://jepsen.io/analyses/mongodb-4.2.6). Nevertheless, Fauna achieved true strong consistency through a novel transaction protocol inspired by [Calvin](https://cs.yale.edu/homes/thomson/publications/calvin-sigmod12.pdf), and was verified by [Jepsen](/assets/pdfs/FaunaDB_2.5.4_Jepsen.pdf). Queries in Fauna were written in FQL, a domain-specific language that focused on safe access and predictable performance from application code. FQL evolved into a Typescript derivative and supported the same complex transactions, constraints, typing, and schema management that SQL databases do, as well as application-level access control and temporality. 

Fauna positioned itself as the first database for the serverless era. Over nearly a decade, it grew from a crude prototype to a high quality, elastic, multi-tenant cloud service, hosted globally and supporting hundreds of customers. Despite raising over $60M in venture capital and achieving essentially all its technical goals, the business side never really came together, and in March 2025, Fauna announced it was shutting down. As a contribution to the community, Fauna [open-sourced](https://faunadb.org) its core database technology so the ideas could live on. Today nearly all cloud databases have serverless provisioning, multi-datacenter replication, and real transactional features, thanks to Fauna showing the way.

## Papers

- **2019-03-05** — [FaunaDB Jepsen Report (PDF)](/assets/pdfs/FaunaDB_2.5.4_Jepsen.pdf)

- **2024-09-13** — [Fauna Architectural Overview (PDF)](/assets/pdfs/Fauna%20Architectural%20Overview.pdf)

## Blog

{% assign blog_posts = site.posts | where: "category", "blog" | sort: "date" %}
{% for post in blog_posts %}
- **{{ post.date | date: "%Y-%m-%d" }}** — [{{ post.title }}]({{ post.url }}) — *{{ post.author }}*
{% endfor %}

## Press

{% assign press_posts = site.posts | where: "category", "press" | sort: "date" %}
{% for post in press_posts %}
- **{{ post.date | date: "%Y-%m-%d" }}** — [{{ post.title }}]({{ post.url }}) — *{{ post.author }}*
{% endfor %}

## Acknowledgements

Thanks to everyone who supported, learned, used, evangelized, funded, debated with, worked at, or contributed in any way to Fauna, and special thanks to Fauna's founding team members: Matt Freels, Brandon Mitchell, Jeff Smick, Erick Pintor, Marrony Neris, Attila Szegedi, Dhruv Gupta, Gayle Grasso, and Ed Ceaser.

So long, and thanks for all the birds. -- _[Evan](https://evanweaver.com/)_


