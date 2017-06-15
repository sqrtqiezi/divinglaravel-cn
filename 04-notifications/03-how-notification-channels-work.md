# How Notification Channels Work

As we discussed earlier, the `Notifications\NotificationSender@sendToNotifiable()` uses the `Notifications\ChannelManager@driver()` method to create a driver based on the channel the notification should be sent through.

The `ChannelManager` extends the `Support\Manager` class which Laravel uses as a factory class in many situations, the `driver()` method has the following:

```php
if (! isset($this->drivers[$driver])) {
    $this->drivers[$driver] = $this->createDriver($driver);
}

return $this->drivers[$driver];
```

The `create()` driver method uses the channel name passed and looks for a method in the ChannelManager class that creates the channel:

```php
$method = 'create'.Str::studly($driver).'Driver';

if (method_exists($this, $method)) {
    return $this->$method();
}
```

So for example if we want to use the slack channel, the manager will be looking for a `createSlackDriver()` method.

Currently there are 5 built in notification channels:

* Database
* Broadcast
* Nexmo SMS
* Slack
* Mail

### Then how does laravel load custom channels?

To pass a custom notification channel you use the channel class name, inside the `ChannelManager` laravel overrides the createDriver() method like this:

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

So in case Laravel couldn't load a driver with the given channel name it'll assume you're trying to load a custom notification channel by its class name, and it uses the container to build an instance of the given class.

## The Database Notification Channel

To send a notification, Laravel calls the `send()` method on that channel, here's how that method looks like in the Database channel:

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

The `routeNotificationFor()` method exists in the `Notifications\RoutesNotifications` trait, this trait is used inside the `Notifications\Notifiable` trait that a User model uses by default in a fresh laravel installation, this method is used to determine where to route the notification to, here's how this method looks like:

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

So it looks for a `routeNotificationFor{DriverName}` method in the notifiable class, and uses the output of such method as the route.

However, for the `mail` driver it uses the `mail` attribute of the notifiable, for the `nexmo` driver it uses the `phone_number` attribute, and for the `database` driver it uses the `notifications()` relationship method by default.

### Where's that method coming from?

The `Notifiable` trait uses the `Notifications\HasDatabaseNotifications` trait which holds the relationship definition between the notifiable model and the DatabaseNotification built-in model.

```php
public function notifications()
{
    return $this->morphMany(DatabaseNotification::class, 'notifiable')
                        ->orderBy('created_at', 'desc');
}
```

### Back to the send method:

What the send method does is that it creates a new database row in the notifications table, it uses a `getData()` internal method to get the content of the notification that will be decoded into JSON before storing it in the database:

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

So it looks for a `toDatabase` or a `toArray` method on the the Notification class and uses the output as the notification data.

## The broadcasting notification channel

Similar to the database channel, it uses a `getData` internal method to get the data of the notification from a `toBroadcast` method or a `toArray` method, however you can return a `Notifications\Messages\BroadcastMessage` from within your `toBroadcast` method, this class implements the `Queueable` trait that you can use to configure how the broadcasted event will be queued.

Here's how the `send` method looks like:

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

So it creates a `BroadcastNotificationCreated` event, and dispatches it using Laravel's Event Dispatcher, as you can see if you return an instance of `BroadcastMessage` you can configure the `queue` and the `connection` of the queued event.

## The Nexmo SMS notification channel

This channel uses `\Nexmo\Client` to send a message using the configuration returned from the `toNexmo()` method of your notification class, in this method you can return a string or an instance of `Notifications\Messages\NexmoMessage`.

If a string was returned it will be used as the SMS body, but you can use the `NexmoMessage` class to have more control over the message:

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

> While creating the NexmoChannel instance inside ChannelManager::createNexmoDriver, Laravel uses configs in your `services.nexmo` configuration keys to define the key/secret of your Nexmo account as well as the default from address.

## Slack notification channel

This channel uses Guzzle to communicate with the Slack message webhook, taking a look at `SlackWebhookChannel::send`:

```php
if (! $url = $notifiable->routeNotificationFor('slack')) {
    return;
}

$this->http->post($url, $this->buildJsonPayload(
    $notification->toSlack($notifiable)
));
```

The `buildJsonPayload` takes care of formatting the output of `toSlack` to a valid webhook payload.

## Mail notification channel

Using the mail notification channel you can build your mail message using a `Mail\Mailable` instance or a `Notifications\Messages\MailMessage`, in your `toMail()` method you can return an instance of `Mailable` and in that case the `MailChannel` will only call `send()` on that instance:

```php
$message = $notification->toMail($notifiable);

if ($message instanceof Mailable) {
    return $message->send($this->mailer);
}
```

On the other hand if you decided to use the Notification System's `MailMessage` the `MailChannel` will use Laravel's `Mail\Mailer` to send the message:

```php
$this->mailer->send($this->buildView($message), $message->data(), function ($mailMessage) use ($notifiable, $notification, $message) {
    $this->buildMessage($mailMessage, $notifiable, $notification, $message);
});
```

By default Laravel uses a default markdown view `notifications::email` to create HTML & Text versions of your message, this allows you to build a simple mail message using PHP only, you can find this markdown view at `Notifications/resources/views/email.blade.php`, it renders the properties of your `MailMessage` instance to build the content of the email:

```php
@foreach ($introLines as $line)
{{ $line }}

@endforeach
```

The `MailMessage` class extends the `SimpleMessage` class that contains a `line()` method, this method fils the `introLines` array with paragraphs you want to show in your mail message, this array is used in the portion of `email.blade.php` shared above to display these paragraphs in your actual email message.

If you take a look at the [official documentation](https://laravel.com/docs/5.4/notifications#mail-notifications) you can find all the configurations you might need to send a mail message, you can also take a look at the `Notifications\Channels\MailChannel` to know how the mail message is built in more detail since we won't discuss mail sending in this dive.