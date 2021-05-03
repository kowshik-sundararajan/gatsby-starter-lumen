---
title: MongoDB Materialized Views
date: "2021-05-02T18:00:00.000Z"
template: "post"
draft: false
slug: "mongodb-materialized-views"
category: "Materialized Views"
tags:
  - "MongoDB"
description: "A brief introduction to MongoDB Materialized Views that can help improve the performance of repetitive aggregate queries."
socialImage: "/media/posts/search.webp"
---

A colleague of mine recently introduced the concept of [materialized views](https://en.wikipedia.org/wiki/Materialized_view "materialized views") in MongoDB to me and I thought it would be fun to write about it. Materialized views are not specific to MongoDB but for the purposes of this article, I won't go beyond the scope of MongoDB.

![database.png](/media/posts/database.png)


### What are materialized views?
Essentially, materialized views are database objects containing the result of a query like the output of a Mongo aggregate query, for example. They come in handy when we want to quickly retrieve the output of aggregate queries joining several collections which are known to be slow in Mongo.

### Use case
Let's say you have a payment gateway product that powers millions of dollars of transactions and you'd like to display a counter of the total amount of transactions to date that has been made on your platform.

One way to do this would be to run an aggregate query every time someone views your counter - this is slow and unnecessary. Instead, you could create a materialized view around the aggregate query and trigger it to run every day. The API can then serve the total amount by performing a simple find query which should be a lot faster.

### How do you create a materialized view?
Materialized views are just like any other mongo collections. In our case, we wanted to create a materialized view to store the output of a complex aggregate query so added the [$merge](https://docs.mongodb.com/manual/reference/operator/aggregation/merge/#mongodb-pipeline-pipe.-merge) stage as the last stage of the aggregate query to insert (first time) or update the collection. Depending on your use case you may want to run the aggregate query weekly, daily or every X seconds to update the materialized view.

Created a mongoose schema for the materialized view:
```javascript
const mongoose = require('mongoose')
const { Schema } = mongoose

const TransactionSummaryMaterializedViewSchema = new Schema(
  {
    totalAmount: {
      type: Number,
      default: 0,
      required: true
    }
  },
  { timestamps: true }
)

module.exports = mongoose.model( 'TransactionSummary', TransactionSummaryMaterializedViewSchema)
```

Aggregate query on the base model

```javavscript
Transactions.aggregate([
// lookups
// unwinds
// ... multiple stages,
{
	$merge: { into: 'TransactionSummary' }
}
])
```

Lastly, the API that would return the counter would perform this find query:
```javascript
TransactionSummary.findOne()
  .then((result) => return result.totalAmount)
```

There you go, a short introduction to MongoDB materialized views and what they can be used for.


## References
- [On-demand Materialized Views](https://docs.mongodb.com/manual/core/materialized-views/)
