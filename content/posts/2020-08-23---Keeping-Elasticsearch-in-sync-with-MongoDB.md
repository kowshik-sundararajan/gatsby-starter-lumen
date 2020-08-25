---
title: Keeping Elasticsearch in sync with MongoDB using Change Streams
date: "2020-08-24T18:00:00.000Z"
template: "post"
draft: false
slug: "keeping-elasticsearch-in-sync-with-mongodb-using-change-streams"
category: "Change Streams"
tags:
  - "MongoDB"
  - "Elasticsearch"
  - "Change streams"
description: "I decided to integrate Elasticsearch for retrieval to improve the search experience on a platform I'm working on. One of the challenges with using two different data stores is keeping them in sync. Updates to MongoDB needed to be synced to Elasticsearch in near real-time."
socialImage: "/media/posts/search.webp"
---

The ability to search and discover things is a must have feature in nearly every platform we use. When developers think about tackling search, the full-text search engine, Elasticsearch, is often the first solution that comes to mind.

![search.webp](/media/posts/search.webp)

The platform that I'm currently working on uses MongoDB for storage and retrieval. Search is optimised using the [`$text`](https://docs.mongodb.com/manual/reference/operator/query/text/#op._S_text) operator. It works well for simple use cases but fails to account for typos, phrases and the overall accuracy and performance did not meet our expectations.

To improve the search experience, I decided to integrate Elasticsearch into the platform for retrieval. One of the challenges with using two different data stores is keeping them in sync. Updates to MongoDB needed to be synced to Elasticsearch in near real-time.

## Mongo Change Streams
I stumbled upon Mongo Change Streams which appeared to fit the use case perfectly. To utilize change streams, we must use a MongoDB replica set, reason being that the change stream works by using the oplog.

My approach was to setup a background worker that listens to events emitted by the change stream and publish them to Elasticsearch.

Opening a change stream against a collection is as simple as:

```javascript
db.collection('person').watch()

Person.watch() // directly using the model

```


### Change Events
A change event typically has the following structure:
```javascript
{
   _id : { <BSON Object> },
   "operationType" : "<operation>",
   "fullDocument" : { <document> },
   "ns" : {
      "db" : "<database>",
      "coll" : "<collection>"
   },
   "to" : {
      "db" : "<database>",
      "coll" : "<collection>"
   },
   "documentKey" : { "_id" : <value> },
   "updateDescription" : {
      "updatedFields" : { <document> },
      "removedFields" : [ "<field>", ... ]
   }
   "clusterTime" : <Timestamp>,
   "txnNumber" : <NumberLong>,
   "lsid" : {
      "id" : <UUID>,
      "uid" : <BinData>
   }
}
```
`_id` contains the event identifier and is used for [resuming change streams](#resuming-a-change-stream)

`fullDocument` contains a point-in-time snapshot of the document.

`documentKey` contains the `_id` of the document in the event.

`operationType` can be one of the following:
- insert: document is inserted
- update: document is updated
- replace: document is replaced
- delete: document is deleted
- drop: collection is dropped
- rename: collection is renamed
- dropDatabase: database is dropped
- invalidate: fired after a drop, rename or dropDatabase event

`update` events by default have the `updateDescription` field which indicates the fields that were updated or removed. If we want to get a point-in-time snapshot of the entire document, we have to specify the `updateLookup: 'fullDocument'` option when opening the change stream.

```javascript
const options = {
  updateDocument: 'fullLookup'
}
Person.watch(options)
```

### Filtering change events
When opening a change stream, we can limit the number of events returned using an aggregation pipeline. This is extremely useful if we only care about certain types of events or data.

Filtering for insert events
```javascript
const pipeline = [
  {
    $match: {
      operationType: 'insert'
    }
  }
]
Person.watch(pipeline)
```

Filtering for events that matches person documents with name = Jane
```javascript
const pipeline = [
  {
    $match: {
      'fullDocument.name': 'Jane'
    }
  }
]
Person.watch(pipeline)
```

### Publishing events to Elasticsearch
Now comes the fun part. The `.watch()` function returns a change stream cursor that we can iterate over to retrieve events. We can then perform transformations on the event and publish them to Elasticsearch.

```javascript
const pipeline = [{ $match: { operationType: 'insert' } }]
const changeStream = Person.watch(pipeline, options)

while (!changeStream.isExhausted()) {
  if (changeStream.hasNext()) {
    const newEvent = changeStream.next()

    // perform ES logic
  }
}
```


### Resuming a change stream
With any technology, chances are they will fail at some point. Building fault-tolerant systems are part and parcel of a developer's life. If a change stream terminates due to any reason, we can resume the stream from a certain point by using either the `startAfter` and `resumeAfter` option.

Every change event has an `_id` which serves as the resume token. We can save this value in a quick access storage system like redis.

```javascript
const options = {
  startAfter: getResumeTokenFromRedis()
}
const changeStream = Person.watch(options)

while (!changeStream.isExhausted()) {
  if (changeStream.hasNext()) {
    const newEvent = changeStream.next()
    // perform ES logic
    saveResumeTokenToCache(newEvent._id)
  }
}
```

Change streams allow us to implement an application built on the [Change Data Capture (CDC)](https://en.wikipedia.org/wiki/Change_data_capture) design pattern where data changes are tracked and events are performed based on the changes.

## References
- [MongoDB Change Streams](https://docs.mongodb.com/manual/changeStreams/)
- [MongoDB Change Events](https://docs.mongodb.com/manual/reference/change-events/)
- [Resuming a change stream](https://docs.mongodb.com/manual/changeStreams/#resume-a-change-stream)

