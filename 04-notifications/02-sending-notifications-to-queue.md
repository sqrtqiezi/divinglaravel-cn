# Sending Notifications To Queue

The simplest way for laravel to queue notifications is to create a single job that sends all the notifications to all the notifiers, but in case one notification failed this would cause the whole Job to be reported as failed even if some notifications were actually sent, and retrying the job would mean that notifications that were already sent successfully will be re-sent again.

To prevent this Laravel creates a queued job for every single notifiable and channel, for example let's say we want to notify 10 customers that the Privacy Policy was updated, and we need to send them an email as well as an SMS message, for that laravel is going to create 20 jobs that sends the notification to 10 clients through 2 different channels.

Laravel also assigns a unique ID for the notification per notifiable, it uses [ramsey/uuid](https://github.com/ramsey/uuid), this unique ID is used when we use the Database channel as the primary key for the notification record, or for when we broadcast the notification.

Long Story short:

* Laravel dispatches a notification to the queue once per channel per notifiable
* Laravel assigns a unique ID to the notification object per notifiable

Inside the `Illuminate\Notifications\NotificationSender` the `queueNotification()` method is called if the notification should be queued, and inside that method laravel dispatches an instance of `Illuminate\Notifications\SendQueuedNotifications` to the queue, this instance is the actual job the worker runs and it holds a reference to the notifiable, notification, channel, and some information about how the notification should be queued which includes:

* The queue connection to be used
* The queue to be used
* Any delay that should be applied before sending

### How can I define these values?

You can set these values as public properties inside the Notification class:

```php
class PolicyUpdateNotification extends Notification implements ShouldQueue
{
    public $connection = 'redis';

    public $queue = 'urgent';

    public $delay = 60;
}
```

Or you can use the methods of the `Illuminate\Bus\Queueable` trait if you use that trait inside your notification.

```php
Notification::send($users, 
    (new TestNotification())->onConnection(...)->onQueue(...)->delay(...)
);
```

### What happens inside the SendQueuedNotifications job?

What happens inside the dispatched job is fairly simple, it calls the `sendNow()` method of the notification manager which sends the notification right away.

It also does some queue-related housekeeping that we won't be looking into in this dive.


[View a chart of the entire process](https://divinglaravel.com/graph/notifications-sequence)