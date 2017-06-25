# 写在前面

Imagine this scenario, as a developer of a large SaaS you're tasked with finding a way to select 10 random customers every minute during the weekend and offer them a discounted upgrade, the job for sending the discount can be pretty easy but we need a way to run it every minute, for that let me share a brief introduction about CRON for those who're not familiar with it.

想象这样一个场景，作为一个大型 SaaS 的开发者，你被分配了一个这样的任务 —— 找到一种方法在周末那天每分钟随机选择 10 个客户并向他们提供优惠升级。发送优惠信息的任务可能非常简单，但我们需要一个方法让这个任务每分钟运行一次。因此，让我分享一个关于 CRON 的简单介绍给那些不熟悉它的人。

## CRON

CRON is a daemon that lives inside your linux server, it's not awake most of the time but every minute it'll open its eyes and see if it's time to run any specific task that was given to it, you communicate with that daemon using crontab files, in most common setups that file can be located at `/etc/crontab`, here's how a crontab file might look like:

CRON 是一个守护进程，它驻留在你的 Linux 服务器中。大部分时间都没有唤醒，但是它每分钟都会打开它的眼睛，查看是否需要运行任何给定的任务。使用 crontab 文件跟这个守护进程通信，在大多数常见设置中，这个文件位于 `/etc/crontab` ，以下是 crontab 文件：

```bash
0 0 1 * * /home/full-backup
0 0 * * * /home/partial-backup
30 5 10 * * /home/check-subscriptions
```

In the crontab file each line represents a scheduled job, and each job definition contains two parts:

1. The * part represents the timer for that job to run.
2. The second part is the command that should run

在 crontab 文件中，每行代表一个调度任务，每个任务包含两部分定义：
1. 符号 * 代表运行这个任务的计时器
2. 第二个部分就是应该运行的命令


### CRON 定时语法

The 5 asterisks represent the following in order:

1. Minute of an hour
2. Hour of a day
3. Day of a month
4. Month of a year
5. Day of a week

五颗星号按照顺序排列如下：

1. 分钟／小时
2. 小时／天
3. 天／月
4. 月／年
5. 天／周

* `0 0 1 * *` in the first example indicates that the job should run on every month, at the first day of the month, at 12 AM, at the first minute of the hour. Or simply it should run every 1st day of the month at 12:00 AM.
* `0 * * * *` in the second example indicates that the job should run every hour.
* `30 5 10 * *` indicates that the job should run on the 10th of every month at 5:30 AM
* `0 0 1 * *` 第一个例子表示该任务应该每个月运行一次，并且是在每个月的第一天，这一天的上午 0 点运行。 或者简单地说，它应该在每个月的第一天上午 00:00 （12:00 AM）时分运行。
* `0 * * * *` 第二个例子表示该任务应该每小时运行一次。
* `30 5 10 * *` 这个表示该任务应该在每个月 10 号上午 5:30 运行

Here are a few other examples:

* `* * * * 3` indicates that the job should run every minute on Wednesdays.
* `* * * * 1-5` indicates that the job should run every minute Monday to Friday.
* `0 1,15 * * *` indicates that the job should run twice a day at 1AM and 3PM.
* `*/10 * * * *` indicates that the job should run every 10 minutes.

以下是其他几个例子：

- `* * * * 3` 表示该任务应该在星期三每分钟运行一次。
- `* * * * 1-5` 表示该任务应该每周一至周五运行。
- `0 1,15 * * *` 表示该任务应该分别在每天上午 1 点和下午 3 点运行。
- `*/10 * * * *` 表示该任务应该每 10 分钟运行一次。

#### 那接下来我们为我们的任务注册一个 cron 任务？

Yeah we can simply register this in our crontab file:

我们可以在 crontab 文件中注册这个：

```bash
 * * * * php /home/divingLaravel/artisan send:offer
```

This command will inform the CRON daemon to run the `php artisan send:offer` artisan command every minute, pretty easy right? But it then gets confusing when we want to run the command every minute only on Thursdays and Tuesdays, or on specific days of the month, having to remember the syntax for cron jobs is not an easy job and also having to update the crontab file every time you want to add a new job or update the schedule can be pretty time consuming sometimes, so a few releases back Laravel added some interesting feature that provides an easy to remember syntax for scheduling tasks:

这个命令会通知 CRON 守护进程每分钟运行一次 artisan 命令： `php artisan send:offer` 。很容易对吗？ 但是，当我们想要只在星期四和星期二或每个特定日子里的每分钟运行命令时，都会感到困惑。你不得不记住cron 任务的语法，另外每次你想添加一个新的工作或更新调度时，你都需要更新 crontab 文件，有时候这可能相当耗时。所以 Laravel 发布添加了一些有趣的功能，为调度任务提供了一个容易记住的语法：

```php
$schedule->command('send:offer')
          ->everyFiveMinutes()
          ->wednesdays();
```

You only have to register one cron jobs in your crontab and laravel takes care of the rest under the hood:

你只需要在你的 crontab 中注册一个 cron 任务，其他的东西 Laravel 会帮你搞定：

```bash
* * * * * php /divingLaravel/artisan schedule:run >> /dev/null 2>&1
```

You may define your scheduled commands inside the schedule method of your `App\Console\Kernel` class:

你可以在 `App\Console\Kernel` 类的 `schedule` 方法中定义调度命令：

```php
protected function schedule(Schedule $schedule)
{
    $schedule->command('send:offer')
          ->everyFiveMinutes()
          ->wednesdays();
}
```

If you'd like more information about the different Timer options, take a look at the [official documentation](http://d.laravel-china.org/docs/5.4).

如果你想了解更多有关不同计时器选项的信息，请查看 [官方文档](http://d.laravel-china.org/docs/5.4/scheduling)。

While the Console Kernel instance is being instantiated, Laravel registers a listener to the Kernel's `booted` event that binds the Scheduler to the container and calls the schedule() method of the kernel:

当 Console 中的 Kernel 实例被实例化时，Laravel 向 Kernel 的 `booted` 事件注册一个监听器，该事件将 Scheduler 绑定到容器并调用 Kernel 的 `schedule()` 方法：

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

一旦终端内核完成 Kernel 类中定义的引导序列，就会触发这个 `booted` 事件。

> Inside the handle() method of the Kernel, Laravel checks if Foundation\Application was booted before, and if not it calls the bootstrapWith() method of the Application and passes the bootstrappers array defined in the console Kernel.
>
> 在 Kernel 的 handle() 方法中，Laravel 会检查 Foundation\Application 是否已启动，如果不是，就调用应用程序的 bootstrapWith() 方法，并传递在 console Kernel 中定义的引导程序数组。

#### 简单的说：

When the CRON daemon calls the `php artisan schedule:run` command every minute, the Console Kernel will be booted up and the jobs you defined inside your `App\Console\Kernel::schedule()` method will be registered into the scheduler.

当 CRON 守护程序每分钟调用 `php artisan schedule:run` 命令时，Console Kernel 将会被启动，你在 `App\Console\Kernel::schedule()` 方法中定义的任务将被注册到调度程序中。

The `schedule()` method takes an instance of `Illuminate\Console\Scheduling\Schedule` as the only argument, this is the schedule manager used to record the jobs you give it and decides what should run every time the CRON daemon pings it.

 `schedule()` 方法采用 `Illuminate\Console\Scheduling\Schedule` 的实例作为唯一的参数，这是用于记录你提供的任务的调度管理器，并决定每次 CRON 守护程序运行的内容。