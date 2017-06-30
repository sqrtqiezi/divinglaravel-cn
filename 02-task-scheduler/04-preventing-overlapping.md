#  防止重叠

Sometimes a scheduled job takes more time to run than what we initially expected, and this causes another instance of the job to start while the first one is not done yet, for example imagine that we run a job that generates a report every minute, after sometime when the data gets huge the report generation might take more than 1 minute so another instance of that job starts while the first is still ongoing.

有时候一个调度任务占用的运行时间会超过我们最初的预期，而这会出现其他任务实例要开始了，第一个都还没有完成。例如，想象我们运行一个需要每分钟生成报告的任务，而当数据量变大时，报告的生成可能需要超过 1 分支的时间，所以就会出现在第一个还在进行的时候就启动这个任务的另一个实例。

In most scenarios this is fine, but sometimes this should be prevented in order to guarantee correct data or prevent a high server resources consumption, so let's see how you can prevent such scenario in laravel:

大多数情况下这并不会造成什么严重的问题，但有时候我们却应该去防止这种情况，以保证正确的数据或者防止大量的服务器资源消耗。接下来让我们看看 Laravel 是如何防止这种情况发生的：

```php
$schedule->command('mail:send')->withoutOverlapping();
```

Laravel will check for the `Console\Scheduling\Event::withoutOverlapping` class property and if it's set to true it'll try to create a mutex for the job, and will only run the job if creating a mutex was possible.

Laravel 会检查 `Console\Scheduling\Event::withoutOverlapping` 类的属性，如果设置为 true，那它会尝试为这个任务创建 mutex，也只有在 mutex 创建成功的情况下这个任务才会运行。

#### But what's a mutex?

但是什么是 mutex？

Here's the most interesting explanation I could find online:

这是我可以在网上找到最有趣的解释：

> When I am having a big heated discussion at work, I use a rubber chicken which I keep in my desk for just such occasions. The person holding the chicken is the only person who is allowed to talk. If you don't hold the chicken you cannot speak. You can only indicate that you want the chicken and wait until you get it before you speak. Once you have finished speaking, you can hand the chicken back to the moderator who will hand it to the next person to speak. This ensures that people do not speak over each other, and also have their own space to talk. Replace Chicken with Mutex and person with thread and you basically have the concept of a mutex.
>
> 当我在工作中进行热烈的讨论时，我会在这样的场合把一只橡胶鸡放在桌子上。 持有鸡的人是唯一被允许说话的人。 如果你没有那只鸡你就不能说话。 你只能指示你想要鸡，直到你可以说话之前才能得到它。 一旦你完成演讲，你可以将鸡还给主持人，他会把这只鸡交给下一个要说话的人。 这样可以确保人们互不交流，也能有自己的说话空间。 同样的思路，把 Mutex 替换成鸡，基本上就是 mutex 的概念。
>
> -- https://stackoverflow.com/questions/34524/what-is-a-mutex/34558#34558

So Laravel creates a mutex when the job starts the very first time, and then every time the job runs it checks if the mutex exists and only runs the job if it doesn't.

所以当工作第一次启动时，Laravel 会创建一个 mutex，然后每次作业运行时，它检查 mutex 是否存在，只有在它不存在的情况下，任务才会运行。

Here's what happens inside the `withoutOverlapping` method:

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

So Laravel creates a filter-callback method that instructs the Schedule Manager to ignore the task if a mutex still exists, it also creates an after-callback that clears the mutex after an instance of the task is done.

因此，如果 mutex 仍然存在，Laravel 会创建一个过滤器回调的方法，通知调度管理器忽略任务，同时它还会创建一个在实例任务完成后清除 mutex 的回调。

Also before running the job, Laravel does the following check inside the `Console\Scheduling\Event::run()` method:

而且在运行任务之前，Laravel 会在 `Console\Scheduling\Event::run()` 方法中进行以下检查：

```php
if ($this->withoutOverlapping && ! $this->mutex->create($this)) {
    return;
}
```

#### Where does the mutex property come from?

#### mutex 属性来自哪里？

While the instance of `Console\Scheduling\Schedule` is being instantiated, laravel checks if an implementation to the `Console\Scheduling\Mutex` interface was bound to the container, if yes it uses that instance but if not it uses an instance of `Console\Scheduling\CacheMutex`:

当 `Console\Scheduling\Schedule` 正在被实例化，Laravel 会检查  `Console\Scheduling\Mutex` 接口的实现是否被绑定到容器，如果是，就使用这个实例，否则就会使用 `Console\Scheduling\CacheMutex` 这个实例。

```php
$this->mutex = $container->bound(Mutex::class)
                        ? $container->make(Mutex::class)
                        : $container->make(CacheMutex::class);
```

Now while the Schedule Manager is registering your events it'll pass an instance of the mutex:

下面的代码显示：调度管理器正在通过传递一个 mutex 实例注册你的事件

```php
$this->events[] = new Event($this->mutex, $command);
```

> By default Laravel uses a cache-based mutex, but you can override that and implement your own mutex approach & bind it to the container.
>
> 默认情况下，Laravel 使用基于缓存的 mutex，但你可以覆盖它并实现自己的 mutex 方法并将其绑定到容器。

### 基于缓存的 mutex

The `CacheMutex` class contains 3 simple methods, it uses the event mutex name as a cache key:

`CacheMutex` 包含了三个简单的方法，它使用了事件 mutex 的名称作为缓存的键：

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

### 任务完成后删除 mutex

As we've seen before, the manager registers an after-callback that removes the mutex after the task is done, for a task that runs a command on the OS that might be enough to ensure that the mutex is cleared, but for a callback task the script might die while executing the callback, so to prevent that an extra fallback was added in `Console\Scheduling\CallbackEvent::run()`:

正如我们之前说的，在任务完成之后管理器注册一个回调来删除 mutex，对于在操作系统上运行的命令坑足以确保 mutex 被清除了，但对于一个回调任务来说，脚本可能在执行回调时中断，所以，为了防止这种事情发生，就在  `Console\Scheduling\CallbackEvent::run()` 中添加了一个额外的回退：

```php
register_shutdown_function(function () {
    $this->removeMutex();
});
```

This removes the mutex in case the script shuts down un-expectedly.

万一脚本意外关闭这会删除 mutex 。