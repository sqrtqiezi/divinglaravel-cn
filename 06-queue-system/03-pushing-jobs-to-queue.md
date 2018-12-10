# Laravel 深入浅出之队列篇（三）

有几种方法可以将任务推入队列：

```php
Queue::push(new InvoiceEmail($order));

Bus::dispatch(new InvoiceEmail($order));

dispatch(new InvoiceEmail($order));

(new InvoiceEmail($order))->dispatch();
```

正如上一节所解释的，`Queue` 门面的调用是对应用程序使用的队列驱动程序的调用。例如，调用 `push` 方法是在使用数据库队列驱动程序时调用 `Queue\DatabaseQueue` 类的 `push` 方法。

有几种方法可以使用：

```php
// 将任务推送到特定队列
Queue::pushOn('emails', new InvoiceEmail($order));

// 在给定的秒数后推送任务
Queue::later(60, new InvoiceEmail($order));

// 将延时任务推送到特定队列
Queue::laterOn('emails', 60, new InvoiceEmail($order));

// 推多个任务
Queue::bulk([
    new InvoiceEmail($order),
    new ThankYouEmail($order)
]);

// 在特定队列上推送多个任务
Queue::bulk([
    new InvoiceEmail($order),
    new ThankYouEmail($order)
], null, 'emails');
```

在调用上面任何一个方法之后，所选队列驱动程序会把给定信息存储在存储空间中，以便 worker 按需提取。

## 使用 bus 命令

使用 bus 命令将任务分配到队列可以提供额外的控制，你可以在任务类中设置所选的 `connection`、`queue` 和  `delay`，来确定命令是否应该排队或立即运行，再或者在运行之前通过管道来发送作业。实际上你甚至可以在你的任务类中处理整个队列任务的执行过程。

`Bus` 门面是 `Contracts\Bus\Dispatcher` 容器的别名，此别名被解析为 `Bus\BusServiceProvider` 中的 `Bus\Dispatcher` 实例：

```php
$this->app->singleton(Dispatcher::class, function ($app) {
    return new Dispatcher($app, function ($connection = null) use ($app) {
        return $app[QueueFactoryContract::class]->connection($connection);
    });
});
```

所以执行 `Bus::dispatch()` 相当于调用 `Bus\Dispatcher` 类中的 `dispatch()` 方法：

```php
public function dispatch($command)
{
    if ($this->queueResolver && $this->commandShouldBeQueued($command)) {
        return $this->dispatchToQueue($command);
    } else {
        return $this->dispatchNow($command);
    }
}
```

这个方法决定是否应该将给定任务分配到队列等候或者立即运行，其中，`commandShouldBeQueued()` 方法检查任务类是否是 `Contracts\Queue\ShouldQueue` 的实例。因此，只有在你的任务类实现  `ShouldQueue`  接口时，才能使用这个方法让任务进入队列。

关于 `dispatchNow` 还是等我们深入讨论 worker 是如何运作的时候，再来讨论吧。现在让我们看看 Bus 命令是如何将任务分派到队列：

```php
public function dispatchToQueue($command)
{
    $connection = isset($command->connection) ? $command->connection : null;

    $queue = call_user_func($this->queueResolver, $connection);

    if (! $queue instanceof Queue) {
        throw new RuntimeException('Queue resolver did not return a Queue implementation.');
    }

    if (method_exists($command, 'queue')) {
        return $command->queue($queue, $command);
    } else {
        return $this->pushCommandToQueue($queue, $command);
    }
}
```

Laravel 会先检查你的任务类中是否定义了 `connection` 属性，你可以设置这个属性告诉 Laravel 它应该将任务发送到哪个队列连接。如果没有定义属性，Laravel 会使用默认的连接。

Laravel 使用 queueResolver 闭包来构建应该使用的队列驱动程序的实例。在注册 Dispatcher 实例时，在 `Bus\BusServiceProvider` 中设置此闭包：

```php
function ($connection = null) use ($app) {
    return $app[Contracts\Queue\Factory::class]->connection($connection);
}
```

`Contracts\Queue\Factory` 是 `Queue\QueueManager` 的别名，也就是说，这个闭包返回 QueueManager 的实例，并为队列管理器设置所需的连接以确定要使用的驱动程序。

最后，`dispatchToQueue` 方法检查任务类是否有 `queue` 方法，如果有，调度程序将只调用此方法，让你完全控制任务应如何排队。你可以选择队列、分配延迟、设置最大重试次数、超时等...

如果没有找到 `queue` 方法，则调用 `pushCommandToQueue()` 会在所选队列驱动程序上调用相应的 `push` 方法：

```php
protected function pushCommandToQueue($queue, $command)
{
    if (isset($command->queue, $command->delay)) {
        return $queue->laterOn($command->queue, $command->delay, $command);
    }

    if (isset($command->queue)) {
        return $queue->pushOn($command->queue, $command);
    }

    if (isset($command->delay)) {
        return $queue->later($command->delay, $command);
    }

    return $queue->push($command);
}
```

调度程序会检查任务类中的 `queue` 和 `delay` 属性，并根据该方法选择适当的队列方法。

#### 所以我可以在作业类中设置队列、延迟和连接？

没错，还可以设置 `tries` 和 `timeout` 属性，队列驱动程序也会使用这些值，所以你的任务类可能如下所示：

```php
class SendInvoiceEmail{
    public $connection = 'default';

    public $queue = 'emails';

    public $delay = 60;

    public $tries = 3;

    public $timeout = 20;
}
```

## 动态设置任务的配置

使用辅助函数 `dispatch()` ，可以执行以下操作：

```php
dispatch(new InvoiceEmail($order))
        ->onConnection('default')
        ->onQueue('emails')
        ->delay(60);
```

上述方法的前提是任务类中使用 `Bus\Queueable` trait ，这个 trait 包含了几种可用于在分配任务之前在任务类上设置属性的方法，例如：

```php
public function onQueue($queue)
{
    $this->queue = $queue;

    return $this;
}
```

#### 但是在你的例子中我们调用了 dispatch() 返回的方法！

这是诀窍：

```php
function dispatch($job)
{
    return new PendingDispatch($job);
}
```

这是 `Foundation/helpers.php` 中 `dispatch()` 辅助函数的定义，它返回一个 `Bus\PendingDispatch` 实例，在这个类里面有这样一个方法：

```php
public function onQueue($queue)
{
    $this->job->onQueue($queue);

    return $this;
}
```

所以当我们执行  `dispatch(new JobClass())->onQueue('default')` 时，`PendingDispatch` 中的 `onQueue` 方法会调用任务类中的 `onQueue` 方法，正如我们前面提到的，任务类需要使用 `Queueable` trait 来处理所有这些执行。

#### 那么调用 Dispatcher::dispatch 方法的部分在哪里？

一旦执行了 `dispatch(new JobClass())->onQueue('default')`，就可以准备好分配任务实例了。实际的工作发生在 `PendingDispatch::__destruct()` 里面：

```php
public function __destruct()
{
    app(Dispatcher::class)->dispatch($this->job);
}
```

调用此方法时，会从容器中解析 `Dispatcher` 的实例，并在其上调用 `dispatch()` 方法。`__destruct()` 是 PHP 的魔术方法，当对该对象的所有引用不再存在或脚本终止时会被调用。因为我们不会在任何地方存储对 `PendingDispatch` 实例的引用，所以 `__destruct` 方法会立即被调用：

```php
// 这里会立即调用析构函数
dispatch(new JobClass())->onQueue('default');

// 这里如果我们调用 unset($temporaryVariable) 会调用析构函数
// 或者当脚本完成执行时。
$temporaryVariable = dispatch(new JobClass())->onQueue('default');
```

### 使用 Dispatchable trait

你可以在任务类上使用 `Bus\Dispatchable` 特征，以便能够像这样分配你的作业：

```php
(new InvoiceEmail($order))->dispatch();
```

`Dispatchable` 的调度方法如下：

```php
public static function dispatch()
{
    return new PendingDispatch(new static(...func_get_args()));
}
```

正如你所看到的，它使用了 `PendingDispatch` 实例，这意味着我们可以这么做：

```php
(new InvoiceEmail($order))->dispatch()->onQueue('emails');
```

