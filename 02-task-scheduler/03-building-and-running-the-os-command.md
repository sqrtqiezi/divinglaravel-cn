# 建立和运行操作系统命令

当启动一个调度事件时，Laravel 的调度管理器 `Illuminate\Console\Scheduling\ScheduleRunCommand` 调用代表该事件的 `Illuminate\Console\Scheduling\Event` 对象上的 `run()` 方法。

 `run()` 方法构建命令语法，并使用 Symfony Process 组件在操作系统上运行。但在构建命令之前，首先检查该命令是否应在后台运行，默认情况下，所有命令都在前台运行，除非在调度命令时使用以下方法：

```php
->runInBackground()
```

#### 什么时候需要在后台运行命令？

想象一下，如果你有几个任务需要同时运行，比如说每小时一次。默认设置下，Laravel 会通知操作系统逐个运行命令：

```bash
~ php artisan update:coupons
# 等待命令完成
# ...
# 命令完成后我们运行下一个
~ php artisan send:mail
```

但是，你可以通知操作系统在后台运行命令，这样即使其他命令尚未完成，你也可以继续推送更多命令：

```bash
~ php artisan update:coupons &
~ php artisan send:mail &
```

在命令末尾加上 ＆符号可以无需等待初始化完成继续推送命令。

`run()` 方法检查 `runInBackground` 属性的值来决定接下来要调用 `runCommandInForeground()` 还是 `runCommandInBackground()`。

如果命令要在前台运行，那剩下的部分就简单些：

```php
$this->callBeforeCallbacks($container);

(new Process(
    $this->buildCommand(), base_path(), null, null, null
))->run();

$this->callAfterCallbacks($container);
```

Laravel 会先执行 `callBeforeCallbacks `，将命令发送到操作系统之后再来执行 `callAfterCallbacks`。

但是，如果命令是在后台运行，Laravel 调用 `callBeforeCallbacks()`，发送命令，但不会调用 `callAfterCallbacks`。因为命令将在后台执行，如果我们此时调用 `callAfterCallbacks()`，它就不会在命令完成后运行，而是命令发送到操作系统，它就运行。

#### 当我们在后台运行命令时，回调会不执行？

照样运行。Laravel 使用下面这样的命令运行后，原来会被完成：

```bash
(php artisan update:coupons ; php artisan schedule:finish eventMutex) &
```

这个命令会导致一系列的两个命令一个接一个地在后台运行。也就是说，当你的 `update:coupons` 命令完成之后，`schedule:finish` 命令会运行给定当前事件的互斥锁，Laravel 定位事件会使用此 ID 并运行其回调。

## 构建命令字符串

当调度程序调用 `runCommandInForeground()` 或 `runCommandInBackground()` 方法时，将调用一个 `buildCommand()` 来构建操作系统将运行的实际命令，该方法只需执行以下操作：

```php
return (new CommandBuilder)->buildCommand($this);
```

要构建命令，需要知道以下配置：

* 命令互斥锁
* 输出应发送到的位置
* 确定输出是否应该附加
* 用户运行命令
* 后台与前台

### 命令互斥锁

互斥锁是为每个命令设置的唯一 ID，Laravel 主要使用它来防止命令重叠（我们稍后再讨论这个）。同时互斥锁也将它用作命令的唯一 ID。

Laravel 在 `Event::mutexName()` 方法中定义了每个命令的互斥锁：

```php
return 'framework'.DIRECTORY_SEPARATOR.'schedule-'.sha1($this->expression.$this->command);
```

也就是说，互斥锁是事件的 CRON 的表达式和命令字符串的组合。

而对于回调事件，互斥锁的创建如下：

```php
return 'framework/schedule-'.sha1($this->description);
```

所以，为了确保你的回调事件设置了正确的互斥锁，你需要为命令设置描述：

```php
$schedule->call(function () {
    DB::table('recent_users')->delete();
})->daily()->description('Clear recent users');
```

### 处理输出

默认情况下，命令的输出会被发送到 `/dev/null` ，这是一个丢弃写入数据的特殊文件。但是，如果要在某个地方发送命令输出，可以在定义命令时使用 `sendOutputTo()` 方法进行更改：

```php
$schedule->command('mail:send')->sendOutputTo('/home/scheduler.log');
```

但这会导致每次的输出都会覆盖任何写入 `scheduler.log` 文件的内容，实现不使用 `appendOutputTo()` 而达到附加输出的作用。命令如下所示：

```php
// 将输出的内容附加到文件
php artisan mail:send >> /home/scheduler.log 2>&1

// 覆盖文件
php artisan mail:send > /home/scheduler.log 2>&1
```

> 2>＆1通知操作系统将错误输出重定向到标准输出通道，简而言之，这意味着错误和输出会被记录到文件之中。

### 使用正确的用户

当你设置用户运行命令时：

```php
$schedule->command('mail:send')->user('forge');
```

Laravel 会运行如下命令：

```php
sudo -u forge -- sh -c 'php artisan mail:send >> /home/scheduler.log 2>&1'
```

### 在后台运行

正如我们之前讨论过的，为了防止命令字符串在后台运行，一般会如下所示：

```bash
(php artisan update:coupons ; php artisan schedule:finish eventMutex) &
```

但这只是一个简短的形式，下面是实际上可以运行的完整的字符串：

```bash
(php artisan update:coupons >> /home/scheduler.log 2>&1 ; php artisan schedule:finish eventMutex) > /dev/null 2>&1  &
```

或者如果你没有将其设置 & 附加输出，它不会定义自定义目标：

```bash
(php artisan update:coupons > /dev/null 2>&1 ; php artisan schedule:finish eventMutex) > /dev/null 2>&1  &
```

### 通过电子邮件发送输出

你可以选择使用 `emailOutputTo()` 方法将命令输出发送到某个电子邮件地址：

```php
$schedule->command('mail:send')->emailOutputTo(['myemail@mail.com', 'myOtheremail@mail.com']);
```

> 如果你只是想收到电子邮件，而不看到输出的内容，可以使用 `emailWrittenOutputTo()`，这样你看到的只是通知这个命令运行的内容。

此方法会更新调度事件的输出属性并将其指向 `storage/logs` 目录中的文件：

```php
if (is_null($this->output) || $this->output == $this->getDefaultOutput()) {
    $this->sendOutputTo(storage_path('logs/schedule-'.sha1($this->mutexName()).'.log'));
}
```

> 注意：只有在尚未设置自定义输出目标位置时，此操作才会起作用。

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
