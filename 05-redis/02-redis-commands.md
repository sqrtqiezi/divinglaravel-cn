# Redis 命令

将产品销售用 key 存储：

```php
Redis::set('product:1:sales', 1000)
Redis::set('product:1:count', 10)
```

现在我们用它读取：

```php
Redis::get('product:1:sales')
```

## 递增和递减计数器

制造新的购买，再增加销售：

```php
Redis::incrby('product:1:sales', 100)

Redis::incr('product:1:count')
```

上面的命令增加 100 个销售，再增加 1 个总数

也可以用同样的方法递减：

```php
Redis::decrby('product:1:sales', 100)

Redis::decr('product:1:count')
```

但当要处理浮点数时，需要使用一个特殊的命令：

```php
Redis::incrbyfloat('product:1:sales', 15.5)

Redis::incrbyfloat('product:1:sales', - 30.2)
```

没有 `decrbyfloat` 命令，但是我们可以将负值传递给 `incrbyfloat` 命令来达到相同的效果。

> incrby、incr、decrby、decr 和 incrbyfloat 操作后的值将作为响应返回

## 检索和更新

现在我们想阅读最新的销售号码，并重新设置计数器为零，可能我们会想在每一天结束时都这样做：

```php
$value = Redis::getset('product:1:sales', 0)
```

这里 `$value` 是保存的 `1000` 的值，如果我们在此操作后读取该值时就会是 `0`。

## Keys 到期

假设我们想在库存不足时向餐厅老板发送通知，但是我们只想每隔 1 小时发送一次通知，而不是每次有新的购买订单都发送通知，所以也许我们设置一个标志，只有当该标志不存在时才发送通知：

```php
Redis::set('user:1:notified', 1, 'EX', 3600);
```

现在这个 key 将在 3600 秒（1小时）后过期，我们可以试着在发送通知之前，检查这个 key 是否存在并设置它：

```php
if(Redis::get('user:1:notified')){
    return;
}

Redis::set('user:1:notified', 1, 'EX', 3600);

Notifications::send();
```

> 注意：不能保证 `user:1:notified` 的值不会在 `get` 和 `set` 操作之间发生变化，接下来将讨论原子命令组，该示例足以让你了解每个命令的工作原理。

我们可以使用以下方式设置 key 的到期时间（以毫秒为单位）：

```php
Redis::set('user:1:notified', 1, 'PX', 3600);
```

并且你也可以使用 `expire` 命令并提供超时秒数：

```php
Redis::expire('user:1:notified', 3600);
```

或以毫秒为单位：

```php
Redis::pexpire('user:1:notified', 3600);
```

如果你希望 key 在特定时间到期，可以使用 `expireat` 并提供 Unix 时间戳：

```php
Redis::expireat('user:1:notified', '1495469730')
```

#### 有什么方法可以检查 key 何时过期？

可以使用 `ttl` 命令（Time To Live），直到 key 到期，它都将返回剩余的秒数。

```php
Redis::ttl('user:1:notified');
```

如果 key 不存在，该命令可能返回 `-2`，如果 key 没有过期设置，则返回 `-1`。

还可以使用 `pttl`命令获取 TTL（以毫秒为单位）。

#### 如果我想取消到期时间怎么办？

```php
Redis::persist('user:1:notified');
```

这个命令会从 key 中删除到期时间，如果成功就返回 `1`，如果 key 不存在或最初没有设置过期，则返回 `0`。

## Keys 存在

假设只有 1 张可以得到的 laracon 票，我们需要关闭这张票的购买，我们可以这样做：

```php
Redis::set('ticket:sold', $user->id, 'NX')
```

这个命令只做 key 设置，如果不存在，下个脚本会尝试将作为 Redis 响应收到的 `null` 设置 key，这意味着该 key 未设置。

还可以指示 Redis 仅在 key 存在时设置 key：

```php
Redis::set('ticket:sold', $user->id, 'XX')
```

如果你想简单地检查一个键是否存在，你可以使用 `exists `命令：

```php
Redis::exists('ticket:sold')
```

## 一次读取多个 key

有时可能需要一次阅读多个 key，可以这样做：

```php
Redis::mget('product:1:sales', 'product:2:sales', 'non_existing_key')
```

这个命令会响应一个与给定 key 大小相同的数组，如果 key 不存在，则其值将为 `null`。

正如我们之前讨论的那样，Redis 以原子方式执行单个命令，这意味着一旦操作开始，没有任何东西可以改变 key 中的任何值，这保证返回的值在读取第一个 key 和最后一个 key 的值时不会改变。

> 使用 mget 更好的原因是启动多个获取命令来减少 RTT（往返时间），即每个命令从客户端到服务器，然后将响应传回给客户端的时间。 更多的内容稍后再说。

## 删除 Keys

使用 `del` 命令一次删除多个 key：

```php
Redis::del('previous:sales', 'previous:return');
```

## 重命名 Keys

使用 `rename` 命令重命名一个 key：

```php
Redis::rename('current:sales', 'previous:sales');
```

如果原始 key 不存在，则返回错误，如果它已经存在，则会重写第二个 key。

重命名键通常是快速操作，除非存在所需名称的键。在这种情况下，Redis 会在重新命名之前先尝试删除现有 key，而删除值非常大的 key 可能会有点慢。

#### 所以必须首先检查第二个键是否存在以防止覆盖？

可以使用 `exists` 来检查第二个键是否存在：

```php
Redis::renamenx('current:sales', 'previous:sales');
```

这首先会检查第二个键是否存在，如果是，它只返回 0 而不做任何事情，因此只有在第二个键不存在的情况下才重命名该键。