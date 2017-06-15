# Before The Dive

Redis is a storage server that persists your data in memory which makes read & write operations very fast, you can also configure it to store the data on disk occasionally, replicate to secondary nodes, and automatically split the data across multiple nodes.

That said, you might want to use Redis when fast access to your data is necessary (caching, live analytics, queue system, etc...), however you'll need another data storage for when the data is too large to fit in memory or you don't really need to have fast access, so a combination of Redis & another relational or non-relational database gives you the power to build large scale applications where you can efficiently store large data but also provide a way to read portions of the data very fast.

## Use Case

Think of a cloud-based Point Of Sale application for restaurants, as owner it's extremely important to be able to monitor different statistics related to sales, inventory, performance of different branches, and many other metrics. Let's focus on one particular metric which is Product Sales, as a developer you want to build a dashboard where the restaurant owner can see live updates on what products have more sales throughout the day.

```sql
select SUM(order_products.price), products.name
from order_products
join products on order_products.product_id = products.id
where DATE(order_products.created_at) = CURDATE()
where order_products.status = "done"
```

The sql query above will bring you a list with the sales each product made throughout the day, but it's a relatively heavy query to run when the restaurant serves thousands of orders every day, simply put you can't run this query in a live-updates dashboard that pulls for updates every 60 seconds or something, it'd be cool if you can cache the product sales every time an order is served and be able to read from the cache, something like:

```php
Event::listen('newOrder', function ($order) {
    $order->products->each(function($product){
        SomeStorage::increment("product:{$product->id}:sales:2017-05-22", $product->sales);
    );
});
```

So now on every new order we'll increment the sales for each product, and we can simply read these numbers later like:

```php
$sales = Product::all()->map(function($product){
    return SomeStorage::get("product:{$product->id}:sales:2017-05-22");
});
```

Replace `SomeStorage` with Redis and now you have today's product sales living in your server's memory, and it's super fast to read from memory, so you don't have to run that large query every time you need to update the analytics numbers in the live dashboard.

## Another Use Case

Now you want to know the number of unique visitors opening your website every day, we might store it in SQL having a table with `user_id` & `date` fields, and later on we can just run a query like:

```sql
select COUNT(Distinct user_id) as count from unique_visits where date = "2017-05-22"
```

So we just have to add a record in that database table every time a user visits the site. But still on a high traffic website adding this extra DB interaction might not be the best idea, wouldn't it be cool if we can just do:

```php
SomeStorage::addUnique('unique_visits:2017-05-22', $user->id);
```

And later on we can do:

```php
SomeStorage::count('unique_visits:2017-05-22');
```

Storing this in memory is fast and reading the stored numbers from memory is super fast, that's exactly what we need, so I hope by now you got a feel on when using Redis might be a good idea. And by the way the method names used in the above examples don't exist in redis, I'm just trying to give you a feel about what you can achieve.

## The Atomic nature of redis operations

Individual commands in Redis are guaranteed to be atomic, that means nothing will change while executing a command, for example:

```php
$monthSales = Redis::getSet('monthSales', 0);
```

This command gets the value of the `monthSales` key and then sets it to zero, it's guaranteed that no other client can change the value or maybe rename the key between the get and set operations, that's due to the single threaded nature of Redis in which a single system process serves all clients at the same time but can only perform 1 operation at a time, it's similar to how you can listen to multiple client alterations on the project at the same time but can only work on 1 alteration at a given moment.

There's also a way to guarantee the atomicity of a group of commands using transactions, more on that later, but briefly let's say you have 2 clients:

```
Client 1 wants to increment the value
Client 1 wants to read that value
Client 2 wants to increment the value
```

These commands might run in the following order:

```
Client 1: Increment value
Client 2: Increment value
Client 1: read value
```

Which will result the read operation from Client 1 to give unexpected results since the value was altered in the middle, that's when a transaction makes sense.
