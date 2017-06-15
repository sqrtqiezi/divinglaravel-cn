# Before The Dive

Laravel is shipped with a Notifications system that makes it super easy to send notifications to your users through different notification channels, here's what a notification object might look like:

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

The `via()` method is used to set the channels Laravel should send the notification through, and you can define multiple methods to customize how the notification should be sent in each channel.

It all starts in `Illuminate\Notifications\ChannelManager` which implements two interfaces:

* `Illuminate\Contracts\Notifications\Dispatcher`
* `Illuminate\Contracts\Notifications\Factory`

You can send notifications using the `Illuminate\Support\Facades\Notification` facade which uses the `ChannelManager` internally:

```php
Notification::send($users, new TestNotification());
```

The `send()` method accepts a single notifiable or an array of notifiables, inside that method Laravel creates an instance of `Illuminate\Notifications\NotificationSender` which handles the actual action of sending the notification, it has three main tasks:

* Prepares the list of notifiables
* Decides if the notification should be queued or sent right away
* Handles the sending/queueying process

The first task is pretty simple, it just formats the given `$notifiables` value into an array of a Collection, that insures the value of notifiables is iteratable for later use.

The second task is simple as well, it checks if the Notification we're passing implements the `Illuminate\Contracts\Queue\ShouldQueue` interface, if so then it means that Notification should be dispatched to queue instead of sending right away.

The third task is where the actual work happens, let's first discover the scenario where a Notification should be queued.

## Sending Notifications right away

If the notification should be sent right away the Channel Manager calls the `sendNow()` method of the `NotificationSender`, this method does the following:

1. Makes sure a notification ID is set
2. Send the notification instance to the different notification drivers/channels
3. Fire a couple of events

First, Laravel fires the `Illuminate\Notifications\Events\NotificationSending`, if any of the listeners to that event returned `false` the notification won't be sent, you can use that to do any final checks.

And after the sending process a `Illuminate\Notifications\Events\NotificationSent` event is fired which you can use to do any logging or housekeeping.

To send the notification, the sender calls the `build()` factory method on the channel manager to build an instance of the channel that should be used and then calls the `send()` method on that channel.

Also I'd like to mention that if you take a look at the `sendNow()` method you'll find that it accepts a third parameter which is the channels that should be used to send the notification, you can use this to simply override the channels specified in the notification class itself, you can actually call `sendNow()` instead of `send()` to make laravel send the notification right away even if the notification class implements the shouldQueue interface:

```php
Notification::sendNow($users, new TestNotification(), ['slack', 'mail']);
```
