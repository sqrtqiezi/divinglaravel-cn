# Redis Hashes

While recording the product sales of a multi-tenant application, we need a way to store the metrics in a way that guarantees a solid separation between each tenant, one idea is to use key names like `shop:{shopId}:product:{productId}:sales` that way we'll have a key per product for each shop, since product IDs might co-exist in multiple shops, we can increment the values of each key on every purchase and get that value when needed, if we need the sales for the whole business we can do something like:

```php
Redis::mget("shop:{$shopId}:product:1", "shop:{$shopId}:product:2", ...);
```

This will bring the sales of every product inside a given business.

### That sounds cool, but seems like you'll introduce a better approach?

I've been reading [this post](https://engineering.instagram.com/storing-hundreds-of-millions-of-simple-key-value-pairs-in-redis-1091ae80f74c?gi=36305aaa4ef0) from the Instagram Engineering blog and I was amazed about the performance gain they described from using Redis Hashes over regular strings, let me share some of the numbers:

* Having 1 Million string keys needed about 70MB of memory
* Having 1000 Hashes each with 1000 Keys only needed 17MB!

The reason behind that is that hashes can be encoded efficiently in a very small memory space, so Redis makers recommend that we use hashes whenever possible since "a few keys use a lot more memory than a single key containing a hash with a few fields", a key represents a Redis Object holds a lot more information than just its value, on the other hand a hash field only hold the value assigned, thus why it's much more efficient.

## Let's build our hash

```php
Redis::hmset("shop:{$shopId}:sales", "product:1", 100, "products:2", 400);
```

This will build a Redis hash with two fields `product:1` and `products:2` holding the values `100` and `400`.

The command `hmset` gives us the ability to set multiple fields of a hash in one go, there's a `hset` command that we can use to set a single field though.

We can read the values of hash fields using the following:

```php
Redis::hget("shop:{$shopId}:sales", 'product:1');
// To return a single value

Redis::hmget("shop:{$shopId}:sales", 'product:1', 'product:2');
// To return values from multiple keys

Redis::hvals("shop:{$shopId}:sales");
// To return values of all fields

Redis::hgetall("shop:{$shopId}:sales");
// Also returns values of all fields
```

In case of `hmget` and `hvals` the return value is an array of values [100, 400], however in case of hgetall the return value is an array of keys & values:

```php
["product:1", 100, "product:2", 400]
```

### Much organized than having multiple keys

Yes and you also stop polluting the key namespace with lots of complex-named keys.

With all the above mentioned benefits there are also a number of useful operations you can do on a hash key:

### Incrementing & Decrementing

```php
Redis:hincrby("shop:{$shopId}:sales", "product:1", 18);
// To increment the sales of product one by 18

Redis:hincrbyfloat("shop:{$shopId}:sales", "product:1", 18.9);
// To increment the sales of product one by 18.9
```

To decrement you just need to provide a negative value, there's no decrby command for hash fields.

### Field Existence

Like string fields you can check if a hash key exists:

```php
Redis::hexists("shop:{$shopId}:sales", "product:1");
```

You can also make sure you don't override an existing field when that's not the desired behaviour:

```php
Redis::hsetnx("shop:{$shopId}:sales", "product:1");
```

This will make sure the field doesn't exist before overriding it.

### Other operations
```php
Redis::hdel("shop:{$shopId}:sales", "product:1", "product:2");
```

This command deletes the given fields from the hash.

```php
Redis::hstrlen("shop:{$shopId}:sales", "product:1");
```

This command returns the string length of the value stored at the given field.

## Performance comes with a cost

As we mentioned before, a hash with a few fields is much more efficient than storing a few keys, a key stores a complete Redis object that contains information about the value stored as well as expiration time, idle time, information about the object reference count, and the type of encoding used internally.

Technically if we create 1 key (Redis Object) that contains multiple string fields it'll require much less memory since every field holds nothing but a reference to the value it holds, and in hashes with small number of fields it's even encoded into a length-prefixed string in a format like:

```php
hashValue = [6]field1[4]val1[6]field2[4]val2
```

Since a hash field holds only a string value we can't associate an expiration time for it, the makers of Redis suggest that we store an individual field to hold the expiration time for each field if need be and get both fields together to compare if the field is still alive:

```php
Redis::hmset('hashKey', 'field1', 'field1_value', 'field1_expiration', '1495786559');
```

So whenever we want to use that key we need to bring the expiration value as well and do the extra work ourselves:

```php
Redis::hmget('hashKey', 'field1', 'field1_expiration');
```

## Some information about encoding hashes

From the Redis docs:

> Hashes, when smaller than a given number of fields, and up to a maximum field size, are encoded in a very memory efficient way that uses up to 10 times less memory (with 5 time less memory used being the average saving). Since this is a CPU / memory trade off it is possible to tune the maximum number of fields and maximum field size.

By default hashes are encoded when they contain less than 512 fields or when the largest values stored in a field is less than 64 in length, but you can adjust these values using the `config` command:

```php
Redis::config('set', 'hash-max-zipmap-entries', 1000);
// Sets the maximum number of fields before the hash stops being encoded

Redis::config('set', 'hash-max-zipmap-value', 128);
// Sets the maximum size of a hash field before the hash stops being encoded
```
