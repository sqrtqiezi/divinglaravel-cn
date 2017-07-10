# Redis 哈希

在记录多用户应用程序的产品销售时，我们需要一种方式来存储指标，以确保每个用户之间的牢固分离。一个方法是使用 key，如 `shop:{shopId}:product:{productId}:sales` ，每个商店的每个产品一个 key。由于产品 ID 可能在多个商店中共存，因此我们可以在每次购买时增加每个 key 的值，并在需要时获取它。如果我们需要整个业务的销售，我们可以执行以下操作：

```php
Redis::mget("shop:{$shopId}:product:1", "shop:{$shopId}:product:2", ...);
```

这将使每个产品的销售在一个给定的业务内。

#### 这听起来很酷，但似乎还有更好的方法？

我从 Instagram Engineering 的博客那里读了 [这篇文章](https://engineering.instagram.com/storing-hundreds-of-millions-of-simple-key-value-pairs-in-redis-1091ae80f74c)，我对使用 Redis Hash 的常规字符串描述的性能提升感到惊讶，让我分享一些数字：

* 拥有 1 百万个字符串 key 需要大约 70MB 的内存
* 1000 个哈希每 1000 个 key 只需要17MB！

背后的原因是可以在非常小的内存空间中有效地编码散列，所以 Redis 的制造者建议我们尽可能使用哈希值，因为「几个 key 比包含几个字段的哈希的单个 key 使用更多的内存」。一个 key 代表一个 Redis 对象比它的值拥有更多的信息。另一方面，散列字段只保留分配的值，这也是为什么它更有效率。

## 构建哈希

```php
Redis::hmset("shop:{$shopId}:sales", "product:1", 100, "products:2", 400);
```

这将构建一个 Redis 哈希，其中包含两个字段 `product:1` 和 `products:2`，保存值为 `100` 和 `400`。

命令 `hmset` 使我们能够一次性设置哈希的多个字段，不过可以使用 `hset` 命令来设置单个字段。

我们可以使用以下内容读取哈希字段的值：

```php
Redis::hget("shop:{$shopId}:sales", 'product:1');
// 返回单个值

Redis::hmget("shop:{$shopId}:sales", 'product:1', 'product:2');
// 从多个 key 返回值

Redis::hvals("shop:{$shopId}:sales");
// 返回所有字段的值

Redis::hgetall("shop:{$shopId}:sales");
// 也返回所有字段的值
```

如果是用 `hmget` 和 `hvals`，返回值是数组值 [100,400]，但是如果是 `hgetall`，返回值是关联数组：

```php
["product:1", 100, "product:2", 400]
```

#### Much organized than having multiple keys

是的，你可以停止用复杂的命名方式污染 key 的命名空间。

利用上述所有优点，还可以在哈希 key 上执行一些有用的操作：

### 递增 & 递减

```php
Redis:hincrby("shop:{$shopId}:sales", "product:1", 18);
// 将产品的销售量增加到 18

Redis:hincrbyfloat("shop:{$shopId}:sales", "product:1", 18.9);
// 将产品的销售额增加 18.9
```

你只需要提供一个负值来做递减，哈希字段没有什么 decrby 命令。

### 字段存在

像字符串字段一样，也可以检查哈希 key 是否存在：

```php
Redis::hexists("shop:{$shopId}:sales", "product:1");
```

也可以确保不覆盖现有字段，但这不是必须的行为：

```php
Redis::hsetnx("shop:{$shopId}:sales", "product:1");
```

这将确保该字段在覆盖之前不存在。

### 其他操作
```php
Redis::hdel("shop:{$shopId}:sales", "product:1", "product:2");
```

这个命令删除哈希中给定的字段。

```php
Redis::hstrlen("shop:{$shopId}:sales", "product:1");
```

这个命令返回存储在给定字段中的值的字符串长度。

## 性能带来成本

正如我们之前提到的，具有几个字段的哈希比存储几个 key 更有效率。一个 key 存储一个完整的 Redis 对象，其中包含的信息有关存储值、到期时间、空闲时间、关于对象引用计数的信息，以及内部使用的编码类型。

从技术上讲，如果我们创建一个包含多个字符串字段的 key（Redis Object），它将需要更少的内存，因为每个字段只保留对它所持有的值的引用。并且在具有少量字段的哈希中，它甚至被编码成一个长度为前缀的字符串，格式如下：

```php
hashValue = [6]field1[4]val1[6]field2[4]val2
```

由于哈希字段只保留一个字符串值，所以我们无法关联它的到期时间。Redis 的制造者们建议我们存储一个单独的字段来保存每个字段的到期时间，如果需要的话，将两个字段一起进行比较（如果该字段仍然存在）。

```php
Redis::hmset('hashKey', 'field1', 'field1_value', 'field1_expiration', '1495786559');
```

所以每当我们想要使用这个 key ，也需要到期值时，要做额外的工作：

```php
Redis::hmget('hashKey', 'field1', 'field1_expiration');
```

## 关于编码哈希的一些信息

Redis 文档：

> 当小于给定数量的字段并且最大字段大小时，哈希值以非常高效的方式进行编码，该方式使用最多 10 倍的内存（使用 5 个时间少的内存是平均保存）。由于这是一个 CPU / 内存权衡，因此可以调整最大字段数和最大字段大小。

By default hashes are encoded when they contain less than 512 fields or when the largest values stored in a field is less than 64 in length, but you can adjust these values using the `config` command:

默认情况下，它们包含少于 512 个字段时编码，或者当字段中存储的最大值小于 64 时，可以使用 `config` 命令调整这些值：

```php
Redis::config('set', 'hash-max-zipmap-entries', 1000);
// 设置哈希停止编码之前的最大字段数

Redis::config('set', 'hash-max-zipmap-value', 128);
// 在哈希停止编码之前设置散列字段的最大值
```
