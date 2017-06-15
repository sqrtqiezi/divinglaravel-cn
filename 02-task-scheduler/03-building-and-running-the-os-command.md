# Building And Running The Os Command

When it's time to fire a scheduled event, Laravel's schedule manager calls the `run()` method on the `Illuminate\Console\Scheduling\Event` object representing that event, this happens inside the `Illuminate\Console\Scheduling\ScheduleRunCommand`.

This `run()` method builds the command syntax and runs it on the operating system using the Symfony Process component, but before building the command it first checks if the command should be running in the background, by default all commands run in the foreground unless you use the following method while scheduling your command:

```php
->runInBackground()
```

### When do I need to run a command in the background?

Imagine if you have several tasks that should run at the same time, say every hour, with the default settings Laravel will instruct the OS to run the commands one by one:

```bash
~ php artisan update:coupons
# Waiting for the command to finish
# ...
# Command finished, now we run the next one
~ php artisan send:mail
```

However, you can instruct the OS to run the commands in the background so that you can continue pushing more commands even if the other ones haven't finished yet:

```bash
~ php artisan update:coupons &
~ php artisan send:mail &
```

Using the ampersand at the end of a command lets you continue pushing commands without having to wait for the initial ones to finish.

The `run()` method checks the value of the `runInBackground` property and decides which method to call next, `runCommandInForeground()` or `runCommandInBackground()`.

In case the command is to be run in the foreground the rest is simple:

```php
$this->callBeforeCallbacks($container);

(new Process(
    $this->buildCommand(), base_path(), null, null, null
))->run();

$this->callAfterCallbacks($container);
```

Laravel executes any before-callbacks, sends the command to the OS, and finally executes any after-callbacks.

However, if the command is to run the background Laravel calls `callBeforeCallbacks()`, sends the command, but doesn't call the after-callbacks, the reason is as you might think, because the command will be executed in the background so if we call `callAfterCallbacks()` at this point it won't be running after the command finishes, it'll run once the command is sent to the OS.

### So no after-callbacks are executed when we run commands in the background?

They run, laravel does that using another command that runs after the original one finishes:

```bash
(php artisan update:coupons ; php artisan schedule:finish eventMutex) &
```

This command will cause a sequence of two commands to run one after another but in the background, so after your `update:coupons` command finishes a `schedule:finish` command will run given the Mutex of the current event, using this ID Laravel locates the event and runs its after-callbacks.

## Building the command string

When the scheduler calls the `runCommandInForeground()` or `runCommandInBackground()` methods, a `buildCommand()` is called to build the actual command that the OS will run, this method simply does the following:

```php
return (new CommandBuilder)->buildCommand($this);
```

To build the command, the following configurations need to be known:

* The command mutex
* The location that output should be sent to
* Determine if the output should be appended
* The user to run the command under
* Background vs Foreground

### The command mutex

A mutex is a unique ID set for every command, Laravel uses it mainly to prevent command overlapping which we will discuss later, but it also uses it as a unique ID for the command.

Laravel defines the mutex of each command inside the `Event::mutexName()` method:

```php
return 'framework'.DIRECTORY_SEPARATOR.'schedule-'.sha1($this->expression.$this->command);
```

So it's a combination of the CRON expression of the event as well as the command string.

However, for callback events the mutex is created as follows:

```php
return 'framework/schedule-'.sha1($this->description);
```

So to ensure having a correct mutex for your callback event you need to set a description for the command:

```php
$schedule->call(function () {
    DB::table('recent_users')->delete();
})->daily()->description('Clear recent users');
```

### Handling output

By default the output of commands is sent to `/dev/null` which is a special file that discards data written to it, however if you want to send the command output somewhere you can change that using the `sendOutputTo()` method while defining the command:

```php
$schedule->command('mail:send')->sendOutputTo('/home/scheduler.log');
```

But this will cause the output to overwrite whatever is written to the `scheduler.log` file every time, to append the output instead you can use `appendOutputTo()`. Here's how the command would look like:

```php
// Append output to file
php artisan mail:send >> /home/scheduler.log 2>&1

// Overwrite file
php artisan mail:send > /home/scheduler.log 2>&1
```

> 2>&1 instructs the OS to redirect error output to the standard output channel, in short words that means errors and output will be logged into your file.

### Using the correct user

When you set a user to run the command:

```php
$schedule->command('mail:send')->user('forge');
```

Laravel will run the command as follows:

```php
sudo -u forge -- sh -c 'php artisan mail:send >> /home/scheduler.log 2>&1'
```

### Running in the background

As we discussed before, the command string will look like this in case it's desired for it to run in the background:

```bash
(php artisan update:coupons ; php artisan schedule:finish eventMutex) &
```

But that's just a short form, here's the complete string that'll actually run:

```bash
(php artisan update:coupons >> /home/scheduler.log 2>&1 ; php artisan schedule:finish eventMutex) > /dev/null 2>&1  &
```

Or this if you don't set it to append output & didn't define a custom destination:

```bash
(php artisan update:coupons > /dev/null 2>&1 ; php artisan schedule:finish eventMutex) > /dev/null 2>&1  &
```

### Sending the output via email

You can choose to send the command output to an email address using the `emailOutputTo()` method:

```php
$schedule->command('mail:send')->emailOutputTo(['myemail@mail.com', 'myOtheremail@mail.com']);
```

> You can also use `emailWrittenOutputTo()` instead if you want to only receive emails if there's an output, otherwise you'll receive emails even if now output for you to see, it'll be just a notification that the command ran.

This method will update the output property of the Scheduled Event and point it to a file in the `storage/logs` directory:

```php
if (is_null($this->output) || $this->output == $this->getDefaultOutput()) {
    $this->sendOutputTo(storage_path('logs/schedule-'.sha1($this->mutexName()).'.log'));
}
```

> Notice that this will only work if you haven't already set a custom output destination.

Next Laravel will register an after-callback that'll try to locate that file, read its content, and send it to the specified recipients.

```php
$text = file_exists($this->output) ? file_get_contents($this->output) : '';

if ($onlyIfOutputExists && empty($text)) {
    return;
}

$mailer->raw($text, function ($m) use ($addresses) {
    $m->to($addresses)->subject($this->getEmailSubject());
});
```
