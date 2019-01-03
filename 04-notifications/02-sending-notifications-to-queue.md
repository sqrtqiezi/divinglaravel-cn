# 用队列发送通知

Laravel 用队列通知最简单的方法是创建一个任务将所有通知发送出去，但是如果其中有一个通知发送失败，就会导致整个任务报告为失败，即使实际上有些通知发送成功了。而重新执行该任务时会将已经发送成功的通知再发送一遍。

为了防止这种情况发生，Laravel 为每个单独的通知和渠道创建了一个队列任务。比方说我们要通知 10 个客户更新隐私政策，向他们发送电子邮件以及短信，因此 Laravel 会创建 20 个任务，即通过两个不同的渠道将通知分别发送到 10 个客户端中去。

Laravel 还为每个通知的客户端用 [ramsey/uuid](https://github.com/ramsey/uuid) 分配唯一 的 ID，这个唯一的 ID 用来当我们使用数据库通道或者用广播通知时作为通知记录的主键。

也就是说：

* Laravel 为每个渠道的每个通知分配一个通知队列
* Laravel 为每个通知的通知对象分配一个唯一的 ID

如果要把通知放进队列池，就调用  `Illuminate\Notifications\NotificationSender` 中的  `queueNotification()` 方法，并且在该方法中，Laravel 将 `Illuminate\Notifications\SendQueuedNotifications` 的实例调度到队列，这个实例是开发者实际运行的任务，它提及了被通知者、通知的内容、渠道跟一些有关通知如何排队的信息，其中包括：

- 要使用的队列连接
- 要使用的队列
- 发送前应用的延迟

#### 如何定义这些值？

你可以将这些值设置为 Notification 类中的公共属性：

```php
class PolicyUpdateNotification extends Notification implements ShouldQueue
{
    public $connection = 'redis';

    public $queue = 'urgent';

    public $delay = 60;
}
```

或者如果你在通知中使用 trait，你可以使用 `Illuminate\Bus\Queueable`  trait 的方法。

```php
Notification::send($users, 
    (new TestNotification())->onConnection(...)->onQueue(...)->delay(...)
);
```

#### 在 SendQueuedNotifications 任务中发生了什么？

分配任务中发生的事情相当简单，它调用了通知管理器中 `sendNow()` 方法来立即发送通知。

它也做了一些队列相关的内务管理，但不在这里深入。
