# 建立和运行操作系统命令

When it's time to fire a scheduled event, Laravel's schedule manager calls the `run()` method on the `Illuminate\Console\Scheduling\Event` object representing that event, this happens inside the `Illuminate\Console\Scheduling\ScheduleRunCommand`.

当启动一个调度事件时，Laravel 的调度管理器 `Illuminate\Console\Scheduling\ScheduleRunCommand` 调用代表该事件的 `Illuminate\Console\Scheduling\Event` 对象上的 `run()` 方法。



This `run()` method builds the command syntax and runs it on the operating system using the Symfony Process component, but before building the command it first checks if the command should be running in the background, by default all commands run in the foreground unless you use the following method while scheduling your command:

 `run()` 方法构建命令语法，并使用 Symfony Process 组件在操作系统上运行。但在构建命令之前，首先检查该命令是否应在后台运行，默认情况下，所有命令都在前台运行，除非在调度命令时使用以下方法：

```php
->runInBackground()
```

#### 什么时候需要在后台运行命令？

Imagine if you have several tasks that should run at the same time, say every hour, with the default settings Laravel will instruct the OS to run the commands one by one:

想象一下，如果有几个任务在每个小时中同时运行。默认设置 Laravel 会通知操作系统逐个运行命令：

```bash
~ php artisan update:coupons
# 等待命令完成
# ...
# 命令完成后我们运行下一个
~ php artisan send:mail
```

However, you can instruct the OS to run the commands in the background so that you can continue pushing more commands even if the other ones haven't finished yet:

但是，你可以通知操作系统在后台运行命令，这样即使其他命令尚未完成，你也可以继续推送更多命令：

```bash
~ php artisan update:coupons &
~ php artisan send:mail &
```

Using the ampersand at the end of a command lets you continue pushing commands without having to wait for the initial ones to finish.

使用命令末尾的＆符号可以继续推送命令，而无需等待初始化完成。



The `run()` method checks the value of the `runInBackground` property and decides which method to call next, `runCommandInForeground()` or `runCommandInBackground()`.

`run()` 方法检查 `runInBackground` 属性的值，并决定下一个调用哪个方法， `runCommandInForeground()` 或 `runCommandInBackground()`。



In case the command is to be run in the foreground the rest is simple:

如果命令要在前台运行，那剩下的部分就简单些：

```php
$this->callBeforeCallbacks($container);

(new Process(
    $this->buildCommand(), base_path(), null, null, null
))->run();

$this->callAfterCallbacks($container);
```

Laravel executes any before-callbacks, sends the command to the OS, and finally executes any after-callbacks.

Laravel 会先执行 `callBeforeCallbacks `，将命令发送到操作系统之后再来执行 `callAfterCallbacks`。



However, if the command is to run the background Laravel calls `callBeforeCallbacks()`, sends the command, but doesn't call the after-callbacks, the reason is as you might think, because the command will be executed in the background so if we call `callAfterCallbacks()` at this point it won't be running after the command finishes, it'll run once the command is sent to the OS.

但是，如果命令是在后台运行，Laravel 调用 `callBeforeCallbacks()`，发送命令，但不会调用后回调，原因是你可能会想，因为命令将在后台执行，所以如果我们调用 `callAfterCallbacks()` 此时它将不会在命令完成后运行，一旦命令发送到操作系统，它将运行。

#### So no after-callbacks are executed when we run commands in the background?

#### 那么当我们在后台运行命令没有回调执行怎么办？

They run, laravel does that using another command that runs after the original one finishes:

照样运行。Laravel 使用另一个命令运行后，原来会被完成：

```bash
(php artisan update:coupons ; php artisan schedule:finish eventMutex) &
```

This command will cause a sequence of two commands to run one after another but in the background, so after your `update:coupons` command finishes a `schedule:finish` command will run given the Mutex of the current event, using this ID Laravel locates the event and runs its after-callbacks.

这个命令会导致一系列的两个命令一个接一个地在后台运行，所以当你的 `update:coupons` 命令完成之后，`schedule:finish` 命令会运行当前事件的 Mutex，Laravel 定位事件会使用此 ID 并运行其回调。

## 构建命令字符串

When the scheduler calls the `runCommandInForeground()` or `runCommandInBackground()` methods, a `buildCommand()` is called to build the actual command that the OS will run, this method simply does the following:

当调度程序调用 `runCommandInForeground()` 或 `runCommandInBackground()` 方法时，将调用一个 `buildCommand()` 来构建操作系统将运行的实际命令，该方法只需执行以下操作：

```php
return (new CommandBuilder)->buildCommand($this);
```

To build the command, the following configurations need to be known:

要构建命令，需要知道以下配置：

* The command mutex
* The location that output should be sent to
* Determine if the output should be appended
* The user to run the command under
* Background vs Foreground
* mutex 命令
* 输出应发送到的位置
* 确定输出是否应该附加
* 用户运行命令
* 后台与前台

### mutex 命令

A mutex is a unique ID set for every command, Laravel uses it mainly to prevent command overlapping which we will discuss later, but it also uses it as a unique ID for the command.

mutex 是每个命令的唯一 ID 集，Laravel 主要使用它来防止重叠命令（我们稍后再讨论这个）同时 mutex 也使用它作为命令的唯一 ID。

Laravel defines the mutex of each command inside the `Event::mutexName()` method:

Laravel 在 `Event::mutexName()` 方法中的每个命令都定义了 mutex  ：

```php
return 'framework'.DIRECTORY_SEPARATOR.'schedule-'.sha1($this->expression.$this->command);
```

So it's a combination of the CRON expression of the event as well as the command string.

也就是事件 CRON 的表达式和命令字符串的组合。

However, for callback events the mutex is created as follows:

而对于回调事件，mutex 的创建如下：

```php
return 'framework/schedule-'.sha1($this->description);
```

So to ensure having a correct mutex for your callback event you need to set a description for the command:

所以为了确保你的回调事件有一个正确的 mutex ，你需要为命令设置一个描述：

```php
$schedule->call(function () {
    DB::table('recent_users')->delete();
})->daily()->description('Clear recent users');
```

### 处理输出

By default the output of commands is sent to `/dev/null` which is a special file that discards data written to it, however if you want to send the command output somewhere you can change that using the `sendOutputTo()` method while defining the command:

默认情况下，命令的输出任务被发送到 `/dev/null` ，这是一个丢弃写入数据的特殊文件，但是如果要在某个地方发送命令输出任务，可以在定义命令时使用 `sendOutputTo()` 方法进行更改：

```php
$schedule->command('mail:send')->sendOutputTo('/home/scheduler.log');
```

But this will cause the output to overwrite whatever is written to the `scheduler.log` file every time, to append the output instead you can use `appendOutputTo()`. Here's how the command would look like:

但这会导致每次的输出都会覆盖任何写入 `scheduler.log` 文件的内容，实现不使用 `appendOutputTo()` 而达到附加输出的作用。命令如下所示：

```php
// 将输出的内容附加到文件
php artisan mail:send >> /home/scheduler.log 2>&1

// 覆盖文件
php artisan mail:send > /home/scheduler.log 2>&1
```

> 2>&1 instructs the OS to redirect error output to the standard output channel, in short words that means errors and output will be logged into your file.
>
> 2>＆1通知操作系统将错误输出重定向到标准输出通道，简而言之，这意味着错误和输出会被记录到文件之中。

### 使用正确的用户

When you set a user to run the command:

当你设置用户运行命令时：

```php
$schedule->command('mail:send')->user('forge');
```

Laravel will run the command as follows:

Laravel 会运行如下命令：

```php
sudo -u forge -- sh -c 'php artisan mail:send >> /home/scheduler.log 2>&1'
```

### 在后台运行

As we discussed before, the command string will look like this in case it's desired for it to run in the background:

正如我们之前讨论过的，为了防止命令字符串在后台运行，一般会如下所示：

```bash
(php artisan update:coupons ; php artisan schedule:finish eventMutex) &
```

But that's just a short form, here's the complete string that'll actually run:

但这只是一个简短的形式，下面是实际上可以运行的完整的字符串：

```bash
(php artisan update:coupons >> /home/scheduler.log 2>&1 ; php artisan schedule:finish eventMutex) > /dev/null 2>&1  &
```

Or this if you don't set it to append output & didn't define a custom destination:

或者如果你没有将其设置为附加输出 &，它不会定义自定义目标：

```bash
(php artisan update:coupons > /dev/null 2>&1 ; php artisan schedule:finish eventMutex) > /dev/null 2>&1  &
```

### 通过电子邮件发送输出

You can choose to send the command output to an email address using the `emailOutputTo()` method:

你可以选择使用 `emailOutputTo()` 方法将命令输出发送到某个电子邮件地址：

```php
$schedule->command('mail:send')->emailOutputTo(['myemail@mail.com', 'myOtheremail@mail.com']);
```

> You can also use `emailWrittenOutputTo()` instead if you want to only receive emails if there's an output, otherwise you'll receive emails even if now output for you to see, it'll be just a notification that the command ran.
>
> 如果有一个输出操作，你想只是收到电子邮件，而不看到输出的内容，你可以使用 `emailWrittenOutputTo()`，这样你看到的只是通知这个命令运行的内容。

This method will update the output property of the Scheduled Event and point it to a file in the `storage/logs` directory:

此方法会更新调度事件的输出属性并将其指向 `storage/logs` 目录中的文件：

```php
if (is_null($this->output) || $this->output == $this->getDefaultOutput()) {
    $this->sendOutputTo(storage_path('logs/schedule-'.sha1($this->mutexName()).'.log'));
}
```

> Notice that this will only work if you haven't already set a custom output destination.
>
> 注意：只有在尚未设置自定义输出目标位置时，此操作才会起作用。

Next Laravel will register an after-callback that'll try to locate that file, read its content, and send it to the specified recipients.

接下来，Laravel 将会注册一个回调去尝试找到该文件，读取其内容，并将其发送给指定的收件人。

```php
$text = file_exists($this->output) ? file_get_contents($this->output) : '';

if ($onlyIfOutputExists && empty($text)) {
    return;
}

$mailer->raw($text, function ($m) use ($addresses) {
    $m->to($addresses)->subject($this->getEmailSubject());
});
```
