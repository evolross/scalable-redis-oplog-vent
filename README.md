# Welcome to the Scalable Redis Oplog
**Note**: This a clone of the original [redis-oplog](https://github.com/cult-of-coders/redis-oplog) to make it scalable


### LICENSE: MIT


## First a Word of Thanks

[Theo](https://github.com/theodorDiaconu) has done the community a great service with redis-oplog. It has become a cornerstone of any major deployment of Meteor. This clone is a major improvement for highly / infinitely scalable Meteor apps. It does have less features (no Vent or SyntheticEvent) as it is optimized for a specific use-case that the original redis-oplog failed to address. We understand we are not the target audience so are grateful for the starting point.

## Problem Statement 
We were facing three major issues with the original redis-oplog

1. We faced major issues with redis-oplog in production on AWS Elatic-Beakstalk, out-of-memory & disconnects from redis. After some research we found that redis-oplog duplicates data (2x for each observer) and re-duplicates for each new observer (even if it's the same collection and same data)
2. DB hits were killing us, each update required multiple hits to update the data then pull it again. This is also another major negative -- not scalable and slow. The approach of keeping pulling from DB to get around the very rare race condition is unsustainable.
3. We want to read from MongoDB secondaries to scale faster. The only way to properly scale with the current redis-oplog is (very) costly sharding.

In addition, the code was becoming complex and hard to understand (which dead code and need for optimization). This is owing to many hands getting involved and its aim to cover as many use cases as possible. **Such an important building-block for us had to be easily maintainable**.

## What we did
This version of redis-oplog is more streamlined (you can see this with the reduced number of settings):

- Uses a single central timed cache at the collection-level, which is also the same place that provides data from when you run `findOne` / `find` -- so full data consistency within the app
- Uses redis to transmit changed fields (we do an actual diff) to other instance caches -- consistency again
- During `update`, we mutate the cache and send the changed fields to the DB and redis -- instead of the current find, update, then find again which has 2 more hits than needed (and is very slow)
- During `insert`, we build the doc and send it via redis to other instances
- During `remove`, we send the ids to be removed and send to other instances
- We use secondary DB reads in our app -- there are potential race conditions in extreme cases which we handle client-side for now; but we are now ready for scalability. If you have more reads --> spin up more secondaries
- Servers can now send data to each other's cache directly via a new feature called 'watchers' (will be documented soon)

In other words, this is not a Swiss-Army knife, it is made for a very specific purpose: **scalable read-intensive real-time application**

## Ideas for future improvements
- Create LUA script for Redis to hold recent history of changes to get around rare race-conditions
- Support external redis publisher [`oplogtoredis`](https://github.com/tulip/oplogtoredis) -- not ready yet for that. I kept the options there but a few things were removed which would break it. Contact me if you need help or are interested in PR. This WILL be done at some point as it is the next level of scalability.
- Create formal Meteor package if there is interest by the community

## Installation


```bash
meteor add disable-oplog
```

In your `<root>/packages` folder
```bash
git clone https://github.com/ramezrafla/redis-oplog.git
meteor add zegenie:redis-oplog
```

**Important**: Make sure `zegenie:redis-oplog` is at the top of your `.meteor/packages` file

Configure it via Meteor settings:

```
// settings.json
{
    ...
    "redisOplog": {}
}

// default full configuration
{
  ...
  "redisOplog": {
    "redis": {
      "port": 6379, // Redis port
      "host": "127.0.0.1" // Redis host
    },
    "retryIntervalMs": 1000, // Retries in 1 second to reconnect to redis if the connection failed
    "mutationDefaults": {
        "optimistic": true, // Does not do a sync processing on the diffs. But it works by default with client-side mutations.
        "pushToRedis": true // Pushes to redis the changes by default
    },
    "cacheTimer": 2700000, // Cache timer, any data not accessed within that time is removed -- our default is 45 mins
    "debug": false, // Will show timestamp and activity of redis-oplog
  }
}
```


```bash
meteor run --settings settings.json
```

## Setup

**Note:** All setup is server-side only, the following methods are not exposed client-side (nor should they be)


### Caching
In your code, for the collections you want to cache (which should really be most of your data):

`collection.startCaching()`

To get hits vs misses you can call the following method from your browser console in **development**

`Meteor.call('__getCollectionStats','myCollectionName',console.log)`

If you want to do this in production, copy the code at the bottom of `/lib/init.js`

This is sample data from our production servers for the `users` collection -- **99% hits!!**:
```
{
  hitRatio: 98.85108236349966
  hits: 6143833
  misses: 71408
}
```

**Note:** If you don't cache, you will still be hitting the DB like in the previous redis-oplog, but slightly better as we strive to use IDs more often in selectors

### Disabling Redis
1. For collections for which you want to skip redis updates entirely (but you can still cache). This is useful for data that is useful for a given user only (in our case analytics collection)

`collection.disableRedis()`

2. For specific mutations

`collection.[update,insert,remove,upsert](<selector>,<modifier>,{pushToRedis:false})`

## Notes

To make sure it is compatible with other packages which extend the `Mongo.Collection` methods, make sure you go to `.meteor/packages`
and put `zegenie:redis-oplog` as the first option.

RedisOplog does not work with _insecure_ package, which is used for bootstrapping your app.

### Events for Meteor (+ Redis Oplog, Grapher and GraphQL/Apollo)

*   Meteor Night 2018 Slide for the original redis-oplog: [Arguments for Meteor](https://drive.google.com/file/d/1Tx9vO-XezO3DI2uAYalXPvhJ-Avqc4-q/view) - Theodor Diaconu, CEO of Cult of Coders: “Redis Oplog, Grapher, and Apollo Live.

## Premium Support

We are here to help. Feel free to contact us at ramez@classroomapp.com for this clone or contact@cultofcoders.com for the original version

## For Developers

The major areas that have seen changes from the original redis-oplog
- `mongo/extendMongoCollection`: Added support for caching
- `mongo/Mutator`: Support for caching, removed sending the whole doc for the removed `protectRaceConditions:false`, check which fields have REALLY changed and only send those, build inserts locally
- `mongo/ObserveMultiplex`: We now call on the cache to get the data, no more local caching of any data, uses projector to send the right fields down
- `cache/ObservableCollection`: No longer caching of data, just IDs; uses cache to build initial adds
- `redis/RedisSubscriptionManager`: Many changes to support using Cache -- removed `getDoc` method
- `redis/WatchManager` and `redis/CustomPublish`: New feature to allow server-server data transfers

Everywhere else, major code cleanups and removing unused helpers in various `/lib` folders

## Contributors

This project exists thanks to all the people who contributed (to the original redis-oplog).
<a href="graphs/contributors"><img src="https://opencollective.com/redis-oplog/contributors.svg?width=890" /></a>

