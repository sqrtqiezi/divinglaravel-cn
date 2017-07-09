# 通知渠道如何工作

正如我们前面讨论的那样, `Notifications\NotificationSender@sendToNotifiable()` 基于通知应该使用的发送渠道的 `Notifications\ChannelManager@driver()` 方法创建驱动程序。

The `ChannelManager` extends the `Support\Manager` class which Laravel uses as a factory class in many situations, the `driver()` method has the following:

`ChannelManager` 继承了 Laravel 在很多情况下使用的工厂类的 `Support\Manager` 类，`driver()` 方法像这样：

```php
if (! isset($this->drivers[$driver])) {
    $this->drivers[$driver] = $this->createDriver($driver);
}

return $this->drivers[$driver];
```

`create()` 驱动方法使用传递渠道名并创建在 ChannelManager 类中查找的方法：

```php
$method = 'create'.Str::studly($driver).'Driver';

if (method_exists($this, $method)) {
    return $this->$method();
}
```

所以如果我们要使用 slack 渠道，那么管理器会寻找一个 `createSlackDriver()` 方法。

目前有 5 个内置通知渠道：

* 数据库
* 广播
* Nexmo 短信
* Slack
* 邮件

#### 那 Laravel 如何加载自定义渠道？

要传递自定义通知渠道，可以使用渠道类名称，在 `ChannelManager` 中，Laravel 会像这样覆盖 `createDriver()` 方法：

```php
try {
    return parent::createDriver($driver);
} catch (InvalidArgumentException $e) {
    if (class_exists($driver)) {
        return $this->app->make($driver);
    }

    throw $e;
}
```

因此，如果 Laravel 无法加载具有给定渠道名称的驱动，则会假设你正尝试通过其类名加载自定义通知渠道，并使用容器来构建给定类的实例。

## 数据库通知渠道

要发送通知，Laravel 会在该渠道上调用 `send()` 方法，以下是数据库渠道中的方法：

```php
public function send($notifiable, Notification $notification)
{
    return $notifiable->routeNotificationFor('database')->create([
        'id' => $notification->id,
        'type' => get_class($notification),
        'data' => $this->getData($notifiable, $notification),
        'read_at' => null,
    ]);
}
```

`routeNotificationFor()` 存在于 `Notifications\RoutesNotifications` trait 中，这个 trait 用于默认情况下 Laravel 安装中使用的 `Notifications\Notifiable` trait。这个方法用于决定路由到哪去，以下是该方法的内容：

```php
public function routeNotificationFor($driver)
{
    if (method_exists($this, $method = 'routeNotificationFor'.Str::studly($driver))) {
        return $this->{$method}();
    }

    switch ($driver) {
        case 'database':
            return $this->notifications();
        case 'mail':
            return $this->email;
        case 'nexmo':
            return $this->phone_number;
    }
}
```

所以它在通知类中寻找一个 `routeNotificationFor{DriverName}` 方法，并使用这种方法的输出作为路由。

However, for the `mail` driver it uses the `mail` attribute of the notifiable, for the `nexmo` driver it uses the `phone_number` attribute, and for the `database` driver it uses the `notifications()` relationship method by default.

而对于 `mail` 驱动，它使用通知的 `mail` 属性。对于 `nexmo` 驱动，它使用 `phone_number` 属性。对于 `database` 驱动，它默认使用 `notifications()` 关系方法。

#### 那个方法来自哪里？

`Notifiable` trait 使用  `Notifications\HasDatabaseNotifications` trait，这个 trait 保存了 notifiable 模型和 DatabaseNotification 内置模型之间的关系定义。

```php
public function notifications()
{
    return $this->morphMany(DatabaseNotification::class, 'notifiable')
                        ->orderBy('created_at', 'desc');
}
```

#### 回到发送方法

send 方法的作用是在通知表中创建一个新的数据库行，它使用一个 `getData()` 内部方法来获取将被解码成 JSON 的通知的内容，然后将其存储在数据库中：

```php
if (method_exists($notification, 'toDatabase')) {
    return is_array($data = $notification->toDatabase($notifiable))
                        ? $data : $data->data;
}

if (method_exists($notification, 'toArray')) {
    return $notification->toArray($notifiable);
}

throw new RuntimeException(
    'Notification is missing toDatabase / toArray method.'
);
```

因此，它在 Notification 类上查找 `toDatabase` 或 `toArray` 方法，并将该输出用作通知数据。

## 广播通知渠道

类似于数据库通道，它使用 `getData` 内部方法从 `toBroadcast` 方法或 `toArray` 方法获取通知的数据，但是你可以从你的 `toBroadcast` 方法中返回 `Notifications\Messages\BroadcastMessage`。此类实现了可以使用 `Queueable` trait 来配置如何将广播事件放进队列。

 `send` 方法如下所示：

```php
$message = $this->getData($notifiable, $notification);

$event = new BroadcastNotificationCreated(
    $notifiable, $notification, is_array($message) ? $message : $message->data
);

if ($message instanceof BroadcastMessage) {
    $event->onConnection($message->connection)
          ->onQueue($message->queue);
}

return $this->events->dispatch($event);
```

所以它创建一个 `BroadcastNotificationCreated` 事件，并使用 Laravel的 事件调度进行分配，你可以看到，如果你返回一个  `BroadcastMessage` 实例，你可以配置 `queue` 和队列事件的 `connection`。

## Nexmo 短信通知渠道

这个渠道使用 `\Nexmo\Client` 从你通知类的 `toNexmo()` 方法返回发送消息的配置，在这种方法中，可以返回一个字符串或 `Notifications\Messages\NexmoMessage` 的实例。

如果返回一个字符串，它将被用作 SMS 主体，但可以使用  `NexmoMessage` 类来更好地控制消息：

```php
if (! $to = $notifiable->routeNotificationFor('nexmo')) {
    return;
}

$message = $notification->toNexmo($notifiable);

if (is_string($message)) {
    $message = new NexmoMessage($message);
}

return $this->nexmo->message()->send([
    'type' => $message->type,
    'from' => $message->from ?: $this->from,
    'to' => $to,
    'text' => trim($message->content),
]);
```

> 在 ChannelManager::createNexmoDriver 内创建 NexmoChannel 实例时，Laravel 使用你的 `services.nexmo` 配置键中的配置来定义 Nexmo 帐户的密钥/密码以及默认地址。

## Slack 通知渠道

该渠道使用 Guzzle 与 Slack webhook 消息通信，看看 `SlackWebhookChannel::send`：

```php
if (! $url = $notifiable->routeNotificationFor('slack')) {
    return;
}

$this->http->post($url, $this->buildJsonPayload(
    $notification->toSlack($notifiable)
));
```

`buildJsonPayload` 负责将 `toSlack` 的输出格式化为有效的 Webhook 负载。

## 邮件通知渠道

使用邮件通知渠道，可以使用 `Mail\Mailable` 实例或 `Notifications\Messages\MailMessage` 构建邮件消息，在 `toMail()` 方法中，可以返回一个 `Mailable` 的实例，在这种情况下， `MailChannel` 只会调用该实例的 `send()`：

```php
$message = $notification->toMail($notifiable);

if ($message instanceof Mailable) {
    return $message->send($this->mailer);
}
```

另一方面，如果决定使用通知系统 `MailMessage`， `MailChannel` 会使用 Laravel 的  `Mail\Mailer`  发送消息：

```php
$this->mailer->send($this->buildView($message), $message->data(), function ($mailMessage) use ($notifiable, $notification, $message) {
    $this->buildMessage($mailMessage, $notifiable, $notification, $message);
});
```

默认情况下，Laravel 使用默认的 markdown 视图 `notifications::email` 来创建消息的 HTML 和 Text 版本，这允许你只使用 PHP 构建简单的邮件消息，你可以在 `Notifications/resources/views/email.blade.php` 找到这个 markdown 视图，它会渲染 `MailMessage` 实例的属性来构建电子邮件的内容：

```php
@foreach ($introLines as $line)
{{ $line }}

@endforeach
```

`MailMessage` 类继承了包含一个 `line()` 方法的 `SimpleMessage` 类，此方法使用要在邮件消息中显示的段落来提交 `introLines` 数组，这个数组用于上面的 `email.blade.php` 部分，以便在实际电子邮件中显示这些段落。

如果查看 [官方文档](https://laravel.com/docs/5.4/notifications#mail-notifications)，可以找到可能需要发送邮件的所有配置。你还可以查看 `Notifications\Channels\MailChannel` 了解邮件内容的详细信息，因为我们不会在此这里中讨论邮件发送。