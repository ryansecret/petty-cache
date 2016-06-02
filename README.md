petty-cache
===========

A cache module for Node.js that uses a two-level cache (in-memory cache for recently accessed data plus Redis for distributed caching) with some extra features to avoid cache stampedes and thundering herds.

Also includes mutex and semaphore distributed locking primitives.

##Features

**Two-level cache**
Data is cached for 2 to 5 seconds in memory to reduce the amount of calls to Redis.

**Jitter**
By default, cache values expire from Redis at a random time between 30 and 60 seconds. This helps to prevent a large amount of keys from expiring at the same time in order to avoid thundering herds (http://en.wikipedia.org/wiki/Thundering_herd_problem).

**Double-checked locking**
Functions executed on cache misses are wrapped in double-checked locking (http://en.wikipedia.org/wiki/Double-checked_locking). This ensures the function called on cache miss will only be executed once in order to prevent cache stampedes (http://en.wikipedia.org/wiki/Cache_stampede).

**Mutex**
Provides a distributed lock (mutex) with the ability to retry a specified number of times after a specified interval of time when acquiring a lock.

**Semaphore**
Provides a pool of distributed locks with the ability to release a slot back to the pool or remove the slot from the pool so that it's not used again.

## Getting Started

```javascript
// Setup petty-cache
var PettyCache = require('petty-cache');
var pettyCache = new PettyCache();

// Fetch some data
pettyCache.fetch('key', function(callback) {
    // This function is called on a cache miss
    fs.readFile('file.txt', callback);
}, function(err, value) {
    // This callback is called once petty-cache has loaded data from cache or executed the specified cache miss function
    console.log(value);
});
```

##API

###new PettyCache([port, [host, [options]]])

Creates a new petty-cache client. `port`, `host`, and `options` are passed directly to [redis.createClient()](https://www.npmjs.org/package/redis#redis-createclient-).

###pettyCache.bulkFetch(keys, cacheMissFunction, [options,] callback)

Attempts to retrieve the values of the keys specified in the `keys` array. Any keys that aren't found are passed to cacheMissFunction as an array along with a callback that takes an error and an object, expecting the keys of the object to be the keys passed to `cacheMissFunction` and the values to be the values that should be stored in cache for the corresponding key.  Either way, the resulting error or key-value hash of all requested keys is passed to `callback`.

**Example**

```javascript
// Let's assume a and b are already cached as 1 and 2
pettyCache.bulkFetch(['a', 'b', 'c', 'd'], function(keys, callback) {
    var results = {};

    keys.forEach(function(key) {
        results[key] = key.toUpperCase();
    }
}, function(err, values) {
    console.log(values); // {a: 1, b: 2, c: 'C', d: 'D'}
});
```

**Options**

```javascript
{
    expire: 30000 // How long it should take for the cache entry to expire in milliseconds. Defaults to a random value between 30000 and 60000 (for jitter).
}
```

###pettyCache.fetch(key, cacheMissFunction, [options,] callback)

Attempts to retrieve the value from cache at the specified key. If it doesn't exist, it executes the specified cacheMissFunction that takes two parameters: an error and a value.  `cacheMissFunction` should retrieve the expected value for the key from another source and pass it to the given callback. Either way, the resulting error or value is passed to `callback`.

**Example**

```javascript
pettyCache.fetch('key', function(callback) {
    // This function is called on a cache miss
    fs.readFile('file.txt', callback);
}, function(err, value) {
    // This callback is called once petty-cache has loaded data from cache or executed the specified cache miss function
    console.log(value);
});
```

**Options**

```javascript
{
    expire: 30000 // How long it should take for the cache entry to expire in milliseconds. Defaults to a random value between 30000 and 60000 (for jitter).
}
```

###pettyCache.get(key, callback)

Attempts to retrieve the value from cache at the specified key. Returns `null` if the key doesn't exist.

**Example**

```javascript
pettyCache.get('key', function(err, value) {
    // `value` contains the value of the key if it was found in the in-memory cache or Redis. `value` is `null` if the key was not found.
    console.log(value);
});
```

###pettyCache.patch(key, value, [options,] callback)

Updates an object at the given key with the property values provided. Sends an error to the callback if the key does not exist.

**Example**

```javascript
pettyCache.patch('key', { a: 1 }, function(callback) {
    if (err) {
        // Handle redis or key not found error
    }

    // The object stored at 'key' now has a property 'a' with the value 1. Its other values are intact.
});
```

**Options**

```javascript
{
    expire: 30000 // How long it should take for the cache entry to expire in milliseconds. Defaults to a random value between 30000 and 60000 (for jitter).
}
```

###pettyCache.set(key, value, [options,] callback)

Unconditionally sets a value for a given key.

**Example**

```javascript
pettyCache.set('key', { a: 'b' }, function(err) {
    if (err) {
        // Handle redis error
    }
});
```

**Options**

```javascript
{
    expire: 30000 // How long it should take for the cache entry to expire in milliseconds. Defaults to a random value between 30000 and 60000 (for jitter).
}
```

## Mutex

###pettyCache.mutex.lock(key, [options, [callback]])

Attempts to acquire a distributed lock for the specified key. Optionally retries a specified number of times by waiting a specified amount of time between attempts.

```javascript
pettyCache.mutex.lock('key', { retry: { interval: 200, times: 3 }, ttl: 1000 }, function(err) {
    if (err) {
        // We weren't able to acquire the lock (even after trying 3 times every 200 milliseconds).
    }

    // We were able to acquire the lock. Do work and then unlock.
    pettyCache.mutex.unlock('key');
});
```

**Options**

```javascript
{
    retry: {
        interval: 200, // The time in milliseconds between attempts to acquire the lock
        times: 1 // The number of attempts to acquire the lock
    },
    ttl: 1000 // The maximum amount of time to keep the lock locked before automatically being unlocked
}
```

###pettyCache.mutex.unlock(key, [callback])

Releases the distributed lock for the specified key.

```javascript
pettyCache.mutex.lock('key', function(err) {
    if (err) {
        // We weren't able to reach Redis. Your lock will expire after its TTL, but you might want to log this error
    }
});
```
