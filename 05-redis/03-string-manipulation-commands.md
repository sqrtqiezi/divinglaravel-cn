# String Manipulation Commands

Imagine we want to keep track of a certain product sales per hour, we could schedule a cron job that runs every hour and does the following:

```php
set("product:1:sales:{$currentHour}", $sales)
```

And then to plot the results later we can use `mget` to retrieve the value of all these keys:

```php
mget('product:1:sales:00', 'product:1:sales:01', ...)
```

And maybe at the beginning of the new day we reset all the keys:

```php
del('product:1:sales:00', 'product:1:sales:01', ...)
```

Ok that might work, but let me show you another way to do it using Redis string manipulation commands:

```php
append("product:1:sales", '00030')
expire("product:1:sales", 86400)
```

Here we'll append the 5-digit sales number every hour to a `product:1:sales` string key, the value of that key after a few hours may look like:

```
000300020000010
```

We also set they key to `expire` after 24 hours automatically using the expire command.

Now since we know that our sales value is always a 5-digit number, we can use the following Redis commands to gather some useful information:

```php
// strlen returns the length of the string

$hoursElapsed = Redis::strlen("product:1:sales") / 5;
```

We can also get the sales in the first 2 hours of the day:

```php
// getrange returns the portion of the string specified by the given offset start and end.
// and str_split will divide the string returned every five characters

$salesUntil2AM = str_split(
    Redis::getrange("product:1:sales", 0, (5 * 2) - 1),
    5
);

// The result will be ['00030', '00200']
```

Or the sales for the whole day:

```php
$sales = str_split(
    Redis::get("product:1:sales"),
    5
);
```

We can also reset the sales at a given hour:

```php
// Here we reset the sales of the second hour, so starting at offset 5 we'll
// overwrite the string for the length of the passed value.

Redis::setrange("product:1:sales", 5, '00300');
```
