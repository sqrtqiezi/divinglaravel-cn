# Before The Dive

Imagine this scenario, as a developer of a large SaaS you're tasked with finding a way to select 10 random customers every minute during the weekend and offer them a discounted upgrade, the job for sending the discount can be pretty easy but we need a way to run it every minute, for that let me share a brief introduction about CRON for those who're not familiar with it.

## CRON

CRON is a daemon that lives inside your linux server, it's not awake most of the time but every minute it'll open its eyes and see if it's time to run any specific task that was given to it, you communicate with that daemon using crontab files, in most common setups that file can be located at `/etc/crontab`, here's how a crontab file might look like:

```bash
0 0 1 * * /home/full-backup
0 0 * * * /home/partial-backup
30 5 10 * * /home/check-subscriptions
```

In the crontab file each line represents a scheduled job, and each job definition contains two parts:

1. The * part represents the timer for that job to run.
2. The second part is the command that should run

## CRON Timing Syntax

The 5 asterisks represent the following in order:

1. Minute of an hour
2. Hour of a day
3. Day of a month
4. Month of a year
5. Day of a week

* `0 0 1 * *` in the first example indicates that the job should run on every month, at the first day of the month, at 12 AM, at the first minute of the hour. Or simply it should run every 1st day of the month at 12:00 AM.
* `0 * * * *` in the second example indicates that the job should run every hour.
* `30 5 10 * *` indicates that the job should run on the 10th of every month at 5:30 AM

Here are a few other examples:

* `* * * * 3` indicates that the job should run every minute on Wednesdays.
* `* * * * 1-5` indicates that the job should run every minute Monday to Friday.
* `0 1,15 * * *` indicates that the job should run twice a day at 1AM and 3PM.
* `*/10 * * * *` indicates that the job should run every 10 minutes.

## So we register a cron task for our job?

Yeah we can simply register this in our crontab file:

```bash
 * * * * php /home/divingLaravel/artisan send:offer
```

This command will inform the CRON daemon to run the `php artisan send:offer` artisan command every minute, pretty easy right? But it then gets confusing when we want to run the command every minute only on Thursdays and Tuesdays, or on specific days of the month, having to remember the syntax for cron jobs is not an easy job and also having to update the crontab file every time you want to add a new job or update the schedule can be pretty time consuming sometimes, so a few releases back Laravel added some interesting feature that provides an easy to remember syntax for scheduling tasks:

```php
$schedule->command('send:offer')
          ->everyFiveMinutes()
          ->wednesdays();
```

You only have to register one cron jobs in your crontab and laravel takes care of the rest under the hood:

```bash
* * * * * php /divingLaravel/artisan schedule:run >> /dev/null 2>&1
```

You may define your scheduled commands inside the schedule method of your `App\Console\Kernel` class:

```php
protected function schedule(Schedule $schedule)
{
    $schedule->command('send:offer')
          ->everyFiveMinutes()
          ->wednesdays();
}
```

If you'd like more information about the different Timer options, take a look at the [official documentation](http://d.laravel-china.org/docs/5.4).

While the Console Kernel instance is being instantiated, Laravel registers a listener to the Kernel's `booted` event that binds the Scheduler to the container and calls the schedule() method of the kernel:

```php
// in Illuminate\Foundation\Console\Kernel

public function __construct(Application $app, Dispatcher $events)
{
    $this->app->booted(function () {
        $this->defineConsoleSchedule();
    });
}

protected function defineConsoleSchedule()
{
     // Register the Scheduler in the Container
    $this->app->instance(
        Schedule::class, $schedule = new Schedule($this->app[Cache::class])
    );

     // Call the schedule() method that we override in our App\Console\Kernel
    $this->schedule($schedule);
}
```

This `booted` event is fired once the console kernel finishes the bootstrapping sequence defined in the Kernel class.

> Inside the handle() method of the Kernel, Laravel checks if Foundation\Application was booted before, and if not it calls the bootstrapWith() method of the Application and passes the bootstrappers array defined in the console Kernel.

## Simply put:

When the CRON daemon calls the `php artisan schedule:run` command every minute, the Console Kernel will be booted up and the jobs you defined inside your `App\Console\Kernel::schedule()` method will be registered into the scheduler.

The `schedule()` method takes an instance of `Illuminate\Console\Scheduling\Schedule` as the only argument, this is the schedule manager used to record the jobs you give it and decides what should run every time the CRON daemon pings it.
