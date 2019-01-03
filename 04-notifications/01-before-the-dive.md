# 写在前面

Laravel 自带通知系统，可以让你超级方便地通过不同的渠道向你的用户发送提醒，一个通知对象看起来就像这样：

```php
class TestNotification extends Notification
{
    public function via($notifiable)
    {
        return ['mail', 'database'];
    }

    public function toMail($notifiable)
    {
        return (new MailMessage)
                    ->line('The introduction to the notification.')
                    ->action('Notification Action', url('/'))
                    ->line('Thank you for using our application!');
    }
}
```

`via()` 方法用于设置 Laravel 发送通知的渠道，而且你也可以定义多种方法来自定义每个渠道该如何发送通知。

所有的一切都在 `Illuminate\Notifications\ChannelManager`  中开始的，它实现了两个接口：

* `Illuminate\Contracts\Notifications\Dispatcher`
* `Illuminate\Contracts\Notifications\Factory`

你可以使用 `ChannelManager` 内部的 `Illuminate\Support\Facades\Notification` facade 来发送通知：

```php
Notification::send($users, new TestNotification());
```

`send()` 方法接受单个或者多个通知数组。在这个方法中，Laravel 创建了一个 `Illuminate\Notifications\NotificationSender` 实例来处理发送通知的实际操作，主要有三个步骤：

* 准备通知列表
* 确认是否应立即加入队列或直接发送通知
* 处理发送/队列进程

第一个任务很简单，它只是将给定的 `$notifiables` 值格式化成一个集合数组，这样可以确保数组的值可以迭代，方便以后使用。

第二个任务也很简单，它检查我们传递的通知是否实现了 `Illuminate\Contracts\Queue\ShouldQueue`  接口，如果是，那么通知会被发送到队列而不是立即发送。

第三个任务是实际工作发生的地方，我们首先发现通知加入队列的场景。

## 立即发送通知

如果通知应立即发送，则 Channel Manager 将调用  `NotificationSender` 的 `sendNow()` 方法，该方法执行以下操作：

1. 确保设置了通知 ID
2. 将通知实例发送到不同的通知驱动/通道
3. 触发几个事件

首先，Laravel 触发 `Illuminate\Notifications\Events\NotificationSending`，如果该事件的监听器返回 `false` ，则不会发送通知。你可以使用这个方法进行最终检查。

发送之后，`Illuminate\Notifications\Events\NotificationSent` 事件被触发，可以使用它来进行日志记录或清理。

发送方在渠道管理器上调用 `build()` 工厂方法构建应该使用的渠道的实例来发送通知，然后在该渠道上调用 `send()` 方法。

另外我想提一下，如果你看一下 `sendNow()` 方法，你会发现这个方法的第三个参数是用来指定通知的渠道，你可以使用它来简单地覆盖通知类本身指定的渠道，实际上即使通知类实现了 shouldQueue 接口，也可以调用 `sendNow()` 来代替 `send()` 使 Laravel 立即发送通知：

```php
Notification::sendNow($users, new TestNotification(), ['slack', 'mail']);
```
