# Redis Commands

Let's store our product sales in a key:

```php
Redis::set('product:1:sales', 1000)
Redis::set('product:1:count', 10)
```

Now to read it we use:

```php
Redis::get('product:1:sales')
```

## Incrementing and Decrementing counters

A new purchase was made, let's increment the sales:

```php
Redis::incrby('product:1:sales', 100)

Redis::incr('product:1:count')
```

Here we increment the sales key by 100, and increment the count key by 1.

We can also decrement in the same way:

```php
Redis::decrby('product:1:sales', 100)

Redis::decr('product:1:count')
```

But when it comes to dealing with floating point numbers we need to use a special command:

```php
Redis::incrbyfloat('product:1:sales', 15.5)

Redis::incrbyfloat('product:1:sales', - 30.2)
```

There's no `decrbyfloat` command, but we can pass a negative value to the `incrbyfloat` command to have the same effect.

> The incrby, incr, decrby, decr, and incrbyfloat return the value after the operation as a response

## Retrieve and update

Now we want to read the latest sales number and reset the counters to zero, maybe we do that at the end of each day:

```php
$value = Redis::getset('product:1:sales', 0)
```

Here `$value` will hold the `1000` value, if we read the value of that key after this operation it'll be `0`.

## Keys Expiration

Let's say we want to send a notification to the owner when inventory is low, but we only want to send that notification once every 1 hour instead of sending it every time a new purchase is made, so maybe we set a flag once and only send the notification when that flag doesn't exist:

```php
Redis::set('user:1:notified', 1, 'EX', 3600);
```

Now this key will expire after 3600 seconds (1 hour), we can check if the key exists before attempting to set it and send the notification:

```php
if(Redis::get('user:1:notified')){
    return;
}

Redis::set('user:1:notified', 1, 'EX', 3600);

Notifications::send();
```

> Notice: There's no guarantee that the value of user:1:notified won't change between the get and set operations, we'll discuss atomic command groups later, but this example is enough for you to understand how every individual command works.

We can set the expiration of a key in milliseconds as well using:

```php
Redis::set('user:1:notified', 1, 'PX', 3600);
```

And you may also use the expire command and provide the timeout in seconds:

```php
Redis::expire('user:1:notified', 3600);
```

Or in milliseconds:

```php
Redis::pexpire('user:1:notified', 3600);
```

And if you want the keys to expire at a specific time you can use expireat and provide a Unix timestamp:

```php
Redis::expireat('user:1:notified', '1495469730')
```

### Is there a way I can check when a key should expire?

You can use the ttl command (Time To Live), which will return the number of seconds remaining until the key expires.

```php
Redis::ttl('user:1:notified');
```

That command may return `-2` if the key doesn't exist, or `-1` if the key has no expiration set.

You can also use the `pttl` command to get the TTL in milliseconds.

### What if I want to cancel expiration?

```php
Redis::persist('user:1:notified');
```

This will remove the expiration from your key, it'll return 1 if OK or 0 if key doesn't exist or originally had no expiration set.

## Keys Existence

Let's say there's only 1 laracon ticket available and we need to close purchasing once that ticket is sold, we can do:

```php
Redis::set('ticket:sold', $user->id, 'NX')
```

This will only set the key if it doesn't exist, the next script that tries to set the key wll receive `null` as a response from Redis which means that the key wasn't set.

You can also instruct Redis to set the key only if it exists:

```php
Redis::set('ticket:sold', $user->id, 'XX')
```

If you want to simply check if a key `exists`, you can use the exists command:

```php
Redis::exists('ticket:sold')
```

## Reading multiple keys in one go

Sometimes you might need to read multiple keys in one go, you can do this:

```php
Redis::mget('product:1:sales', 'product:2:sales', 'non_existing_key')
```

The response of this command is an array with the same size of the given keys, if a key doesn't exist its value is going to be `null`.

As we discussed before, Redis executes individual command atomically, that means nothing can change the value of any in the keys once the operation started, so it's guaranteed that the values returned are not altered in between reading the value of the first key and the last key.

> Using mget is better that firing multiple get commands to reduce the RTT (round trip time), which is the time each individual command takes to travel from client to the server and then carry the response back to the client. More on that later.

## Deleting Keys

You can also delete multiple keys at once using the del command:

```php
Redis::del('previous:sales', 'previous:return');
```

## Renaming Keys

You can rename a key using the `rename` command:

```php
Redis::rename('current:sales', 'previous:sales');
```

An error is returned if the original key doesn't exist, and it overrides the second key if it already exists.

Renaming keys is usually a fast operation unless a key with the desired name exists, in that case Redis will try to delete that existing key first before renaming this one, deleting a key that holds a very big value might be a bit slow.

### So I have to check first if the second key exists to prevent overriding?

Yeah you can use `exists` to check if the second key exists... OR:

```php
Redis::renamenx('current:sales', 'previous:sales');
```

This will check first if the second key exists, if yes it just returns 0 without doing anything, so it only renames the key if the second key does not exist.
