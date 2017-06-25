# 事件的属性

Every entry you add is converted into an instance of `Illuminate\Console\Scheduling\Event` and stored in an `$events` class property of the Scheduler, an Event object consists of the following:

你添加的每个条目都将转换为 `Illuminate\Console\Scheduling\Event` 的实例，并存储在 Scheduler 的 `$events` 类属性中，Event 对象由以下内容组成：

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
* 命令运行
* CRON 表达式
* 用于评估时间的时区
* 操作系统用户的运行命令
* 命令应该运行的环境列表
* 维护模式配置
* 事件重叠配置
* 命令前台/后台运行配置
* 用于决定该命令是否运行的检查列表
* 如何处理输出的配置
* 命令运行后运行的回调函数
* 命令运行前运行的回调函数
* 命令说明
* 唯一的 Mutex 命令

The command to run could be one of the following:

运行的命令有：

* A callback
* A command to run on the operating system
* An artisan command
* A job to be dispatched
* 回调
* 在操作系统上运行的命令
* artisan 命令
* 要派遣的任务

### 使用回调

In case of a callback, the `Container::call()` method is used to run the value we pass which means we can pass a callable or a string representing a method on a class:

在回调的情况下，使用 `Container::call()` 方法来运行我们传递的值，这意味着我们可以传递一个可调用或一个表示方法的字符串：

```php
protected function schedule(Schedule $schedule)
{
    $schedule->call(function () {
        DB::table('recent_users')->delete();
    })->daily();
}
```

或者：

```php
protected function schedule(Schedule $schedule)
{
    $schedule->call('MetricsRepository@cleanRecentUsers')->daily();
}
```

### 传递操作系统命令

If you would like to pass a command for the operating system to run you can use `exec()`:

如果要传递操作系统的命令来运行，可以使用 `exec()`：

```php
$schedule->exec('php /home/sendmail.php --user=10 --attachInvoice')->monthly();
```

还可以以数组的形式传递参数：

```php
$schedule->exec('php /home/sendmail.php', [
    '--user=10',
    '--subject' => 'Reminder',
    '--attachInvoice'
])->monthly();
```

### 通过 artisan 命令

```php
$schedule->command('mail:send --user=10')->monthly();
```

你也可以传递类名：

```php
$schedule->command('App\Console\Commands\EmailCommand', ['user' => 10])->monthly();
```

The values you pass are converted under the hood to an actual shell command and passed to `exec()` to run it on the operating system.

传递的值在底层转换为实际的 shell 命令，并传递给 `exec()` 以在操作系统上运行它。

### 分配任务

You may dispatch a job to queue using the Job class name or an actual object:

使用 Job 类名或实际对象将任务分配到队列中：

```php
$schedule->job('App\Jobs\SendOffer')->monthly();

$schedule->job(new SendOffer(10))->monthly();
```

Under the hood Laravel will create a callback that calls the `dispatch()` helper method to dispatch your command.

在底层，Laravel 会创建一个回调，调用 `dispatch()` 辅助方法来分配命令。

So the two actual methods of creating an event here is by calling `exec()` or `call()`, the first one submits an instance of `Illuminate\Console\Scheduling\Event` and the latter submits `Illuminate\Console\Scheduling\CallbackEvent` which has some special handling.

所以在这里创建一个事件，通过调用两个实际方法 `exec()` 或 `call()`，前者提交了 `Illuminate\Console\Scheduling\Event` 的一个实例，后者提交了带特殊的处理的 `Illuminate\Console\Scheduling\CallbackEvent`。

## 构建 cron 表达式

Using the timing method of the Scheduled Event, laravel builds a CRON expression for that event under the hood, by default the expression is set to run the command every minute:

使用调度事件的计时方法，Laravel 会为该事件创建一个 CRON 表达式。默认情况下，表达式设置为每分钟运行一次命令：

```bash
* * * * * *
```

But when you call `hourly()` for example the expression will be updated to:

但是，当你调用 `hourly()` 时，表达式会被更新为：

```bash
0 * * * * *
```

If you call `dailyAt('13:30')` for example the expression will be updated to:

如果你调用 `dailyAt('13:30')` ，表达式会更新为：

```bash
30 13 * * * *
```

If you call `twiceDaily(5, 14)` for example the expression will be updated to:

如果你调用 `twiceDaily(5, 14)` ，表达式会被更新为：

```bash
0 5,14 * * * *
```

A very smart abstraction layer that saves you tons of research to find the right cron expression, however you can pass your own expression if you want as well:

一个非常聪明的抽象层，可以节省大量的研究来找到正确的 cron 表达式，但是如果你想要的话也可以传递你自己的表达式：

```bash
$schedule->command('mail:send')->cron('0 * * * * *');
```

#### 那时区呢？

If you want the CRON expression to be evaluated with respect to a specific timezone you can do that using:

如果你希望 CRON 表达式针对特定时区进行评估，则可以使用以下方式进行：

```bash
->timezone('Europe/London')
```

Under the hood Laravel checks the timezone value you set and update the `Carbon` date instance to reflect that.

在底层，Laravel 会检查你设置的时区值，并更新 `Carbon` 日期实来以反映这一点。

#### 那么 Laravel 会用 CRON 表达式检查命令是否到期？

Exactly, Laravel uses the [mtdowling/cron-expression](https://github.com/mtdowling/cron-expression) library to determine if the command is due based on the current system time (with respect to the timezone we set).

实际上，Laravel 使用 [mtdowling/cron-expression](https://github.com/mtdowling/cron-expression) 库来确定命令是否基于当前系统时间（相对于我们设置的时区）。

## 在运行命令时添加约束

### 区间约束

For example if you want the command to run daily but only between two specific dates:

例如，如果希望命令每天运行，但只是在两个特定日期之间运行：

```php
->between('2017-05-27', '2017-06-26')->daily();
```

And if you want to prevent it from running during a specific period:

或者如果你想防止它在一段特定的时间内运行：

```php
->unlessBetween('2017-05-27', '2017-06-26')->daily();
```

### 环境约束

You can use the `environments()` method to pass the list of environments the command is allowed to run under:

你可以使用 `environments()` 方法传递命令允许运行的环境列表：

```php
->environments('staging', 'production');
```

### 维护模式

By default scheduled commands won't run when the application is in maintenance mode, however you can change that by using:

默认情况下，当应用程序处于维护模式时，调度的命令不会运行，但是你可以通过使用以下命令来更改：

```php
->evenInMaintenanceMode()
```

### OS 用户

You can set the Operating System user that'll run the command using:

使用以下命令设置运行命令的操作系统用户：

```php
->user('forge')
```

Under the hood Laravel will use `sudo -u forge` to set the user on the operating system.

在底层中，Laravel 将使用 `sudo -u forge` 在操作系统上设置用户。

### 自定义约束

You can define your own custom constraint using the `when()` and `skip()` methods:

使用 `when()` 和 `skip()` 方法自定义约束：

```php
// 仅当用户数大于 1000 时，才能运行该命令
->when(function(){
    return User::count() > 1000;
});

// 运行命令，除非用户数大于 1000
->skip(function(){
    return User::count() > 1000;
});
```

## 回调前后

Using the `before()` and `then()` methods you can register callbacks that'll run before or after the command finishes execution:

使用 `before()` 和 `then()` 方法，可以注册命令执行之前或执行完之后运行的回调：

```php
->before(function(){
    Mail::to('myself@Mail.com', new CommandStarted());
})
->then(function(){
    Mail::to('myself@Mail.com', new CommandFinished());
});
```

You can also ping URLs or webhooks using the `pingBefore()` and `thenPing()` methods:

还可以使用 `pingBefore()` 和 `thenPing()` 方法 ping URL 或 webhooks：

```php
->ping('https://my-webhook.com/start')->thenPing('https://my-webhook.com/finish')
```

Using these commands laravel registers a before/after callbacks under the hood and uses Guzzle to send a `GET` HTTP request:

使用这些命令，Laravel 会在底层注册之前／之后的回调，并使用 Guzzle 发送一个 `GET` 的 HTTP 请求：

```php
return $this->before(function () use ($url) {
    (new HttpClient)->get($url);
});
```
