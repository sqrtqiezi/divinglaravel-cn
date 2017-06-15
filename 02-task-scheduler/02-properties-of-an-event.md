# Properties Of An Event

Every entry you add is converted into an instance of `Illuminate\Console\Scheduling\Event` and stored in an `$events` class property of the Scheduler, an Event object consists of the following:

* Command to run
* CRON Expression
* Timezone to be used to evaluate the time
* Operating System User the command should run as
* The list of Environments the command should run under
* Maintenance mode configuration
* Event Overlapping configuration
* Command Foreground/Background running configuration
* A list of checks to decide if the command should run or not
* Configuration on how to handle the output
* Callbacks to run after the command runs
* Callbacks to run before the command runs
* Description for the command
* A unique Mutex for the command

The command to run could be one of the following:

* A callback
* A command to run on the operating system
* An artisan command
* A job to be dispatched

## Using a callback

In case of a callback, the `Container::call()` method is used to run the value we pass which means we can pass a callable or a string representing a method on a class:

```php
protected function schedule(Schedule $schedule)
{
    $schedule->call(function () {
        DB::table('recent_users')->delete();
    })->daily();
}
```

Or:

```php
protected function schedule(Schedule $schedule)
{
    $schedule->call('MetricsRepository@cleanRecentUsers')->daily();
}
```

## Passing a command for the operating system

If you would like to pass a command for the operating system to run you can use `exec()`:

```php
$schedule->exec('php /home/sendmail.php --user=10 --attachInvoice')->monthly();
```

You can also pass the parameters as an array:

```php
$schedule->exec('php /home/sendmail.php', [
    '--user=10',
    '--subject' => 'Reminder',
    '--attachInvoice'
])->monthly();
```

## Passing an artisan command

```php
$schedule->command('mail:send --user=10')->monthly();
```

You can also pass the class name:

```php
$schedule->command('App\Console\Commands\EmailCommand', ['user' => 10])->monthly();
```

The values you pass are converted under the hood to an actual shell command and passed to `exec()` to run it on the operating system.

## Dispatching a Job

You may dispatch a job to queue using the Job class name or an actual object:

```php
$schedule->job('App\Jobs\SendOffer')->monthly();

$schedule->job(new SendOffer(10))->monthly();
```

Under the hood Laravel will create a callback that calls the `dispatch()` helper method to dispatch your command.

So the two actual methods of creating an event here is by calling `exec()` or `call()`, the first one submits an instance of `Illuminate\Console\Scheduling\Event` and the latter submits `Illuminate\Console\Scheduling\CallbackEvent` which has some special handling.

## Building the cron expression

Using the timing method of the Scheduled Event, laravel builds a CRON expression for that event under the hood, by default the expression is set to run the command every minute:

```bash
* * * * * *
```

But when you call `hourly()` for example the expression will be updated to:

```bash
0 * * * * *
```

If you call `dailyAt('13:30')` for example the expression will be updated to:

```bash
30 13 * * * *
```

If you call `twiceDaily(5, 14)` for example the expression will be updated to:

```bash
0 5,14 * * * *
```

A very smart abstraction layer that saves you tons of research to find the right cron expression, however you can pass your own expression if you want as well:

```bash
$schedule->command('mail:send')->cron('0 * * * * *');
```

### How about timezones?

If you want the CRON expression to be evaluated with respect to a specific timezone you can do that using:

```bash
->timezone('Europe/London')
```

Under the hood Laravel checks the timezone value you set and update the `Carbon` date instance to reflect that.

### So laravel checks if the command is due using the CRON expression?

Exactly, Laravel uses the [mtdowling/cron-expression](https://github.com/mtdowling/cron-expression) library to determine if the command is due based on the current system time (with respect to the timezone we set).

## Adding Constraints on running the command

### Duration constraints

For example if you want the command to run daily but only between two specific dates:

```php
->between('2017-05-27', '2017-06-26')->daily();
```

And if you want to prevent it from running during a specific period:

```php
->unlessBetween('2017-05-27', '2017-06-26')->daily();
```

### Environment constraints

You can use the `environments()` method to pass the list of environments the command is allowed to run under:

```php
->environments('staging', 'production');
```

### Maintenance Mode

By default scheduled commands won't run when the application is in maintenance mode, however you can change that by using:

```php
->evenInMaintenanceMode()
```

### OS User

You can set the Operating System user that'll run the command using:

```php
->user('forge')
```

Under the hood Laravel will use `sudo -u forge` to set the user on the operating system.

### Custom Constraints

You can define your own custom constraint using the `when()` and `skip()` methods:

```php
// Runs the command only when the user count is greater than 1000
->when(function(){
    return User::count() > 1000;
});

// Runs the command unless the user count is greater than 1000
->skip(function(){
    return User::count() > 1000;
});
```

### Before and After callbacks

Using the `before()` and `then()` methods you can register callbacks that'll run before or after the command finishes execution:

```php
->before(function(){
    Mail::to('myself@Mail.com', new CommandStarted());
})
->then(function(){
    Mail::to('myself@Mail.com', new CommandFinished());
});
```

You can also ping URLs or webhooks using the `pingBefore()` and `thenPing()` methods:

```php
->ping('https://my-webhook.com/start')->thenPing('https://my-webhook.com/finish')
```

Using these commands laravel registers a before/after callbacks under the hood and uses Guzzle to send a `GET` HTTP request:

```php
return $this->before(function () use ($url) {
    (new HttpClient)->get($url);
});
```
