# 字符串操作命令

想像一下，我们想跟踪每小时的某一产品销售，我们可以安排一个每小时运行一次的 cron 作业，并执行以下操作：

```php
set("product:1:sales:{$currentHour}", $sales)
```

然后为了绘制结果，我们可以使用 `mget` 来检索所有这些 key 的值：

```php
mget('product:1:sales:00', 'product:1:sales:01', ...)
```

也许我们需要在新的一天的开始时重置所有的 key：

```php
del('product:1:sales:00', 'product:1:sales:01', ...)
```

这也是一种方法，但让我告诉你另一种使用 Redis 字符串操作命令的方式：

```php
append("product:1:sales", '00030')
expire("product:1:sales", 86400)
```

在这里，我们将每小时将 5 位数的销售数字追加到 `product:1:sales`  字符串 key 中，几个小时后这个 key 的值可能如下所示：

```
000300020000010
```

我们还设置 key 在 24 小时后自动使用 `expire` 命令。

现在，由于知道销售值是一个 5 位的数字，所以可以使用以下 Redis 命令来收集一些有用的信息：

```php
// strlen 返回字符串的长度

$hoursElapsed = Redis::strlen("product:1:sales") / 5;
```

也可以在一天的头两个小时获得销售额：

```php
// getrange 返回由给定的偏移开始和结束指定的字符串的部分。
// str_split 将分隔每五个字符返回的字符串

$salesUntil2AM = str_split(
    Redis::getrange("product:1:sales", 0, (5 * 2) - 1),
    5
);

// 返回结果会是 ['00030', '00200']
```

或全天的销售：

```php
$sales = str_split(
    Redis::get("product:1:sales"),
    5
);
```

也可以在给定时间重置销售额：

```php
// 这里我们重置第二个小时的销售，所以从偏移量 5 开始我们会覆盖传递值的长度的字符串。
Redis::setrange("product:1:sales", 5, '00300');
```
