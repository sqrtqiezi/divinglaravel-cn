# 事件的属性

你添加的每个条目都将转换为 `Illuminate\Console\Scheduling\Event` 的实例，并存储在 Scheduler 的 `$events` 类属性中，Event 对象由以下内容组成：

* 命令运行
* CRON 表达式
* 用于评估时间的时区
* 操作系统用户的运行命令
* 命令应该运行的环境列表
* 维护模式配置
* 事件重叠配置
* 命令前台/后台运行配置
* 用于确定命令是否应运行的检查列表
* 如何处理输出的配置
* 命令运行后运行的回调函数
* 命令运行前运行的回调函数
* 命令的说明
* 唯一的 Mutex 命令

要运行的命令可以是以下之一：

* 回调
* 在操作系统上运行的命令
* artisan 命令
* 要调度的任务

### 使用回调

在回调的情况下，使用 `Container::call()` 方法来运行我们传递的值，这意味着我们可以传递一个可调用的或一个表示类上方法的字符串：

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

如果要传递操作系统的命令，可以使用 `exec()`：

```php
$schedule->exec('php /home/sendmail.php --user=10 --attachInvoice')->monthly();
```

以数组的形式传递参数：

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

传递的值在底层转换为实际的 shell 命令并传递给 `exec()` 运行。

### 分配任务

使用任务类的名称或实际对象将任务分配到队列中：

```php
$schedule->job('App\Jobs\SendOffer')->monthly();

$schedule->job(new SendOffer(10))->monthly();
```

Laravel 会在底层创建一个回调来调用 `dispatch()` 辅助方法分配命令。

所以在这里创建的事件是通过调用两个实际方法 `exec()` 或 `call()`，前者提交了 `Illuminate\Console\Scheduling\Event` 的一个实例，后者提交了带特殊的处理的 `Illuminate\Console\Scheduling\CallbackEvent`。

## 构建 cron 表达式

使用调度事件的计时方法，Laravel 会为该事件创建一个 CRON 表达式。默认情况下，表达式设置为每分钟运行一次命令：

```bash
* * * * * *
```

但是，当你调用 `hourly()` 时，表达式会被更新为：

```bash
0 * * * * *
```

如果你调用 `dailyAt('13:30')` ，表达式会更新为：

```bash
30 13 * * * *
```

如果你调用 `twiceDaily(5, 14)` ，表达式会被更新为：

```bash
0 5,14 * * * *
```

这用了一个非常聪明的抽象层，可以为你节省大量的研究怎么写正确的 cron 表达式。但是如果你想要自己写的话也可以传递你自己的表达式：

```bash
$schedule->command('mail:send')->cron('0 * * * * *');
```

#### 那时区呢？

如果要根据特定时区评估 CRON 表达式，可以使用以下方式进行评估：

```bash
->timezone('Europe/London')
```

Laravel 会在底层检查你设置的时区值，并更新 `Carbon` 日期实来响应这一点。

#### 那么 Laravel 会用 CRON 表达式检查命令是否到期？

实际上，Laravel 使用 [mtdowling/cron-expression](https://github.com/mtdowling/cron-expression) 库来确定命令是否基于当前系统时间（相对于我们设置的时区）。

## 在运行命令时添加约束

### 区间约束

例如，如果想要命令只在两个特定日期之间每天运行：

```php
->between('2017-05-27', '2017-06-26')->daily();
```

或者如果你想防止它在一段特定的时间内运行：

```php
->unlessBetween('2017-05-27', '2017-06-26')->daily();
```

### 环境约束

你可以使用 `environments()` 方法传递允许命令运行的环境：

```php
->environments('staging', 'production');
```

### 维护模式

默认情况下，当应用程序处于维护模式时，调度的命令不会运行，但是你可以通过使用以下命令来更改：

```php
->evenInMaintenanceMode()
```

### OS 用户

使用以下命令设置运行命令的操作系统用户：

```php
->user('forge')
```

Laravel 会在底层使用 `sudo -u forge` 在操作系统上设置用户。

### 自定义约束

使用 `when()` 和 `skip()` 方法自定义约束：

```php
// 用户数大于 1000 时，才会运行命令
->when(function(){
    return User::count() > 1000;
});

// 用户数大于 1000 时，命令不会运行
->skip(function(){
    return User::count() > 1000;
});
```

## 回调前后

使用 `before()` 和 `then()` 方法，可以注册命令执行之前或执行完之后运行的回调：

```php
->before(function(){
    Mail::to('myself@Mail.com', new CommandStarted());
})
->then(function(){
    Mail::to('myself@Mail.com', new CommandFinished());
});
```

还可以使用 `pingBefore()` 和 `thenPing()` 方法 ping URL 或 webhooks：

```php
->ping('https://my-webhook.com/start')->thenPing('https://my-webhook.com/finish')
```

使用这些命令，Laravel 会在底层注册前／后回调，并使用 Guzzle 发送 HTTP `GET` 请求：

```php
return $this->before(function () use ($url) {
    (new HttpClient)->get($url);
});
```

