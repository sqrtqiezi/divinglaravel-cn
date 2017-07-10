# 写在前面

Redis 是一种存储服务器，可以将数据保留在内存中，使得读写操作可以非常快。还可以将其配置成数据偶尔存储磁盘上，复制到辅助节点，并在多个节点间自动分割数据。

也就是说，你可能希望在快速访问数据时使用 Redis（缓存、实时分析、队列系统等），然而，当数据太大而不能放进内存时，就需要用另一个数据存储，或者你不需要快速访问。因此，Redis 和另一个关系数据库或非关系数据库的组合使你有能力构建大型应用程序，可以高效地存储大量数据，同时也提供了一种非常快速地读取部分数据的方法。

## 用例

在一个基于云的销售点销售应用程序的餐厅，作为餐厅老板，监控与销售、库存、不同分支机构的业绩以及许多其他指标相关的不同统计数据是非常重要的。我们专注于产品销售的一个特定指标，作为开发人员，你想要建立一个仪表板，餐厅老板可以随时查看更多销售额的实时更新。

```sql
select SUM(order_products.price), products.name
from order_products
join products on order_products.product_id = products.id
where DATE(order_products.created_at) = CURDATE()
where order_products.status = "done"
```

上面的 sql 查询提供每个产品全天销售的列表，但是当餐厅每天服务数以千计的订单时，运行的时间就比较多。简单地说，你不能每 60 秒或者实时在实时更新的仪表板中运行此查询。如果你可以在每次增加一张订单时缓存产品销售量，并且从缓存中读取，就会很酷，如下所示：

```php
Event::listen('newOrder', function ($order) {
    $order->products->each(function($product){
        SomeStorage::increment("product:{$product->id}:sales:2017-05-22", $product->sales);
    );
});
```

所以现在每增加一个新的订单就会增加每个产品的销售额，我们可以简单地阅读这些数字，如：

```php
$sales = Product::all()->map(function($product){
    return SomeStorage::get("product:{$product->id}:sales:2017-05-22");
});
```

使用 Redis 替换 `SomeStorage`，今天的商品销售已经存在你的服务器内存里，从内存中读取数据的速度时非常快的，因此每当你需要更新实时仪表板中的分析数据时，就不需要再运行上面的查询了。

## 另一用例

现在你想知道每天打开你的网站的访问者的数量，我们可能会将其存储在具有  `user_id` 和 `date` 字段的表的 SQL 中，稍后我们可以运行一个查询，如：

```sql
select COUNT(Distinct user_id) as count from unique_visits where date = "2017-05-22"
```

So we just have to add a record in that database table every time a user visits the site. But still on a high traffic website adding this extra DB interaction might not be the best idea, wouldn't it be cool if we can just do:

因此，每当用户访问该站点时，我们只需在该数据库表中添加一条记录。但对于在一个高流量的网站来说，添加这个额外的数据库交互可能不是一个好主意：

```php
SomeStorage::addUnique('unique_visits:2017-05-22', $user->id);
```

接下来我们可以做：

```php
SomeStorage::count('unique_visits:2017-05-22');
```

将其存储在内存中是快速的，从内存读取存储的数字是超快的，这正是我们需要的，所以我希望现在你有一个感觉，就是使用 Redis 可能是一个好主意。顺便一提，上述示例中使用的方法名在 redis 中是不存在的，我只是想给你一个可以实现的逻辑。

## Redis 操作的原子性

Redis 中的单个命令保证是原子的，这意味着在执行命令时没有任何改变，例如：

```php
$monthSales = Redis::getSet('monthSales', 0);
```

此命令获取 `monthSales` 键的值，然后将其设置为零。确保没有其他客户端可以更改值或者在获取和设置操作之间重命名密钥，这是由于 Redis 的单一线程特性，其中单个系统进程同时为所有客户端服务，但一次只能执行 1 个操作。它类似于你可以同时监听项目中的多个客户端更改，但只能在给定时刻进行 1 次更改。

还有一种方法，使用事务来保证一组命令的原子性，再简单地说，你有两个客户端：

```
客户端 1 想要增加值
客户端 1 想读这个值
客户端 2 想要增加值
```

这些命令可能按以下顺序运行：

```
客户端 1: 增加值
客户端 2: 增加值
客户端 1: 读取值
```

这将导致客户端 1 的读取操作产生意想不到的结果，因为值在中间被更改，这交易有意义的。