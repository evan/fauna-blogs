---
layout: post
title: "Time-Traveling Databases: Exploring Temporality in Fauna"
date: 2016-10-17
description: "Is a temporal database the same as a time-series database? This is a frequent point of confusion. What is a time series? Let’s start by discussing the more popular concept of the time-series database."
original_url: "https://fauna.com/blog/time-traveling-databases"
author: "Daniel Abadi, Matt Freels"
category: blog
---

⚠️ Disclaimer ⚠️

### This post refers to a previous version of FQL.

This post refers to a previous version of FQL (v4). For the most current version of the language, visit our FQL documentation.

Is a temporal database the same as a time-series database? This is a frequent point of confusion.

## What is a time series?

Let’s start by discussing the more popular concept of the time-series database.

Time-series databases are optimized for low-latency recording of values that change over time: often sampled data like temperature, stock price, or fuel consumption. They are also useful for counting and aggregating events: number of cars that pass over a road, number of votes cast, number of likes on a social media post. As such they are optimized heavily for writing numeric data.

To achieve this, time-series databases support only very simple transaction patterns that do not involve multiple keys, types other than numbers, large data sizes, or indexes. This is fine. They make it easy to study aggregated trends across time periods. Complex business transactions remain the domain of operational databases, and complex analysis remains the domain of columnar or map/reduce analytics systems.

> Time-series databases support only very simple transaction patterns.

## What is temporality?

The concept behind a temporal database is more subtle. Rather than recording sampled numeric data ordered by time, a temporal database tracks every change to business data within a retention period. In other words, it is a historical database. As transactions are processed, they append, rather than overwrite, previous state. The previous state of the world can be viewed by running a complex query with a timestamp in the past.

## Temporality in Fauna

Fauna is a temporal database, not a time-series database. That said, Fauna can be a great solution for many time-series use cases involving values that change over time, for example, data center operational metrics.

In Fauna, all records (including schema) are temporal and support configurable retention policies. When records are updated or deleted, their prior contents are not overwritten; instead, a new immutable version at the current transaction timestamp is inserted into the instance history, either as a create, update, or delete event. All transactions, including transactions involving indexes, can be executed at a point in the past (or the future!), or transformed into a change feed of the events between any two points in time.

> In Fauna, all transactions can be executed at any point in the past or transformed into a change feed.

This is extremely useful for auditing business transactions, undoing developer mistakes or security breaches–even deleting an entire database can be reversed–, syncing partially-connected clients like mobile phones, constructing activity feeds, keeping analytics systems up-to-date and is a fundamental part of Fauna’s isolation model.

### Instance History

One way temporality in Fauna makes developers’ lives easier is through ‘snapshots’. Imagine that you need to ask questions about the state of an entity at a specific time, or within a date range. For example, you are building a ‘Friend Locator’ app. Users check in to update their current location, which results in the database setting a field on the user instance:

```
update(ref(class('users'), 123), params: { data: { location: 'Sydney' } })
```

```
{
  "ref": { "@ref": "classes/users/123" },
  "ts": <clock_time>,
  "data": {
    "location": "Sydney"
  }
}
```

Want to show the user where the user was at the same time last week? Simply retrieve the user record using a timestamp in the past:

```
get(ref(class('users'), 123), ts: <week ago="">)</week>
```

```
{
  "ref": { "@ref": "classes/users/123" },
  "ts": <week_ago>,
  "data": {
    "location": "San Francisco"
  }
}
```

### Index History

Or, since Fauna maintains temporality even in indexes, you can query an index for where all of a user’s friends were in the past:

```
paginate(match(index('friends_by_location'), ref(class('users'), 123)))
```

```
{
  "data": [
    ["Austin", { "@ref": "classes/users/789" }],
    ["Los Angeles", { "@ref": "classes/users/234" }],
    ["New York", { "@ref": "classes/users/456" }],
    ["Oakland", { "@ref": "classes/users/567" }]
  ]
}
```

```
paginate(match(index('friends_by_location'), ref(class('users'), 123)), ts: <week ago="">)</week>
```

```
{
  "data": [
    ["Chicago", { "@ref": "classes/users/456" }],
    ["Fremont", { "@ref": "classes/users/789" }],
    ["Houston", { "@ref": "classes/users/567" }],
    ["San Diego", { "@ref": "classes/users/234" }]
  ]
}
```

### Change Feeds

If you want to provide the user a journal view of where the user has been recently, Fauna’s events view takes temporality beyond snapshots. The events view returns a change feed of how data the result set changed over time:

```
map(paginate(ref(class('users'), 123), after: <week_ago>, events: true)) do |event|
  get(select('resource', event), select('ts', event))
end
```

```
  "data": [
    {
      "ref": { "@ref": "classes/users/123" },
      "ts": <week_ago>,
      "data": {
        "location": "San Francisco"
      }
    },
    {
      "ref": { "@ref": "classes/users/123" },
      "ts": <day_ago>,
      "data": {
        "location": "Melbourne"
      }
    },
    {
      "ref": { "@ref": "classes/users/123" },
      "ts": <minute_ago>,
      "data": {
        "location": "Sydney"
      }
    }
  ]
}
```

## Conclusion

Time-series databases only store a sequence of numeric values. They cannot respond to queries more sophisticated than a simple list or aggregation of the numeric values they store.

Temporal databases encode temporality into transactional query engines. For this reason, they are vastly more powerful and general purpose than time-series databases. In fact, you can easily create a time-series database within a temporal database by doing rollup aggregations. (Although Fauna does not currently support aggregations natively, rollup aggregations are very simple to implement on the application side.)

> Temporal databases are vastly more powerful and general purpose than time-series databases.

This may bring to mind multi-version concurrency control (MVCC) in SQL systems, like PostgreSQL. It should! MVCC originally began as a specific application of general temporality. But hardware constraints at the time discouraged users from retaining more data than the bare minimum and the feature fell by the wayside.

Those constraints don’t exist in the cloud era. So rather than garbage-collecting old data immediately, Fauna leaves retention policy up to you and your business. Further, neither Postgres nor MySQL database enables you to get a view of the minimal history it does maintain. Fauna not only exposes the history, it integrates it with the full power of the query engine.

## Upcoming Posts

We look forward to providing more sophisticated examples of temporality in the future, as well as discussing in detail how the consistency model also takes advantage of Fauna’s temporal architecture. We will talk about these subjects in detail in forthcoming blog posts.