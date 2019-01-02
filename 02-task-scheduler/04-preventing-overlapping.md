#  防止重叠

有时候一个调度任务占用的运行时间会超过我们最初的预期，而这会出现其他任务实例要开始了，第一个都还没有完成。例如，假设我们运行一个需要每分钟生成报告的任务，而当数据量变大时，报告的生成可能需要超过 1 分钟的时间，所以就会出现在第一个还在进行的时候就启动这个任务的另一个实例。

大多数情况下这并不会造成什么严重的问题，但我们应该去防止这种情况的发生，以保证数据正确或者防止大量的服务器资源被消耗。且让我们看看 Laravel 是如何防止这种情况发生的：

```php
$schedule->command('mail:send')->withoutOverlapping();
```

Laravel 会检查 `Console\Scheduling\Event::withoutOverlapping` 类的属性，如果设置为 true，那它会为这个任务创建互斥锁，并且只有在互斥锁创建成功的情况下这个任务才会运行。

#### 什么是互斥锁？

这是我可以在网上找到最有趣的解释：

> 当我在工作中进行热烈的讨论时，我会在这样的场合把一只橡胶鸡放在桌子上。 持有鸡的人是唯一被允许说话的人。 如果你没有那只鸡你就不能说话。 你只能指示你想要鸡，直到你可以说话之前才能得到它。 一旦你完成演讲，你可以将鸡还给主持人，他会把这只鸡交给下一个要说话的人。 这样可以确保人们互不交流，也能有自己的说话空间。 同样的思路，把互斥锁替换成鸡，基本上就是互斥锁的概念。
>
> -- https://stackoverflow.com/questions/34524/what-is-a-mutex/34558#34558

所以当工作第一次启动时，Laravel 会创建一个互斥锁，然后每次作业运行时，它检查互斥锁是否存在，只有在它不存在的情况下，任务才会运行。

以下是 `withoutOverlapping` 方法中发生的情况：

```php
public function withoutOverlapping()
{
    $this->withoutOverlapping = true;

    return $this->then(function () {
        $this->mutex->forget($this);
    })->skip(function () {
        return $this->mutex->exists($this);
    });
}
```

因此，如果互斥锁仍然存在，Laravel 会创建一个过滤器回调的方法，通知调度管理器忽略任务，同时它还会创建一个在实例任务完成后清除互斥锁的回调。

而且在运行任务之前，Laravel 会在 `Console\Scheduling\Event::run()` 方法中进行以下检查：

```php
if ($this->withoutOverlapping && ! $this->mutex->create($this)) {
    return;
}
```

#### 互斥锁属性来自哪里？

当 `Console\Scheduling\Schedule` 正在被实例化，Laravel 会检查  `Console\Scheduling\Mutex` 接口的实现是否被绑定到容器，如果是，就使用这个实例，否则就会使用 `Console\Scheduling\CacheMutex` 这个实例。

```php
$this->mutex = $container->bound(Mutex::class)
                        ? $container->make(Mutex::class)
                        : $container->make(CacheMutex::class);
```

下面的代码显示：调度管理器正在通过传递一个互斥锁实例注册你的事件

```php
$this->events[] = new Event($this->mutex, $command);
```

> 默认情况下，Laravel 使用基于缓存的互斥锁，但你可以覆盖它并实现自己的互斥锁方法并将其绑定到容器。

### 基于缓存的互斥

`CacheMutex` 包含了三个简单的方法，它使用了事件互斥锁的名称作为缓存的键：

```php
public function create(Event $event)
{
    return $this->cache->add($event->mutexName(), true, 1440);
}

public function exists(Event $event)
{
    return $this->cache->has($event->mutexName());
}

public function forget(Event $event)
{
    $this->cache->forget($event->mutexName());
}
```

### 任务完成后删除互斥锁

正如我们之前说的，在任务完成之后管理器注册一个回调来删除互斥锁。对于在操作系统上，运行的命令足以确保互斥锁被清除了，但对于一个回调任务来说，脚本可能在执行回调时中断，所以，为了防止这种事情发生，就在  `Console\Scheduling\CallbackEvent::run()` 中添加了一个额外的回退：

```php
register_shutdown_function(function () {
    $this->removeMutex();
});
```

万一脚本意外关闭这会执行删除互斥锁的操作。