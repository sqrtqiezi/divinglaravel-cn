# 报告异常

能够跟踪在应用程序中遇到的异常，对于我们了解出现的错误是非常有用的。无需任何配置，所有的异常都会被记录在 `storage/logs/laravel.log` 文件中，并包含全部堆栈踪迹。因此，在任何时间点你都可以通过查看这个文件，来观察你的应用程序在运行过程中是否发生过什么错误。

#### 但有时我并不关心是否抛出某些异常

Laravel 正在考虑这点，它会在报告任何异常之前，检查你是否真的想要让这个异常被报告。事实上，在默认情况下，Laravel 不会报告下面这些类型的异常：

* 认证和授权异常
* 模型未找到异常
* Token 不匹配异常
* Validation 异常
* HTTP 异常
* 响应异常

你也可以自定义一组你不想被 `App\Exceptions\Handler` 类报告的异常，找到 `$dontReport` 属性并且以数组的形式提供一个列表：

```php
class Handler extends ExceptionHandler
{
    protected $dontReport = [
        MyCustomException::class
    ];
}
```

#### 内置错误报告

Laravel 附带了四种不同的错误模式，这些模式是建立在 [Monolog](https://github.com/Seldaek/monolog) 库之上的：

1. 单个日志文件：在 `storage/logs/laravel.log` 文件中记录异常
2. 每日日志文件：在 `storage/logs` 中每天创建日志文件来记录异常
3. syslog 处理程序： 在 syslog 中记录异常。[点击了解更多](https://community.rapid7.com/community/insightops/blog/2017/05/23/what-is-syslog)
4. 错误日志处理程序：在 PHP 错误日志中记录异常

默认情况下 Laravel 为我们选择了单个文件模式，但你也可以通过改变 `config/app.php["log"]` 的配置来切换成任何模式。但是如果要使用了每日日志模式的话，要注意一点，就是 Laravel 会删除旧的日志文件，即只保存最近 5 天的日志，你可以通过改变 `log_max_files` 配置来设置这些日志应该保留多少天。

你也可以在返回 `$app` 实例之前，通过在 `bootstrap/app.php` 文件中调用 `configureMonologUsing` 方法来使用不同的 Monolog 处理程序：

```php
$app->configureMonologUsing(function ($monolog) {
    $monolog->pushHandler(
        new SlackWebhookHandler($webhookUrl)
    );
});

return $app;
```

>如果你想更深入地了解如何配置这些，请查看 `Illuminate\Log\LogServiceProvider`，这是创建记录器并配置 Monolog 的地方。

#### 如果我想用不同的方式报告异常呢？

有两种选择：

- 在 `App\Exceptions\Handler` 类的 `report()` 方法中，你可以整合你自己的报告机制，任何应该被报告的异常都会调用这个方法。这是一个自定义报告的好地方。
- 或者你可以在自定义异常中定义一个 `report()` 来定义你自己的报告机制。

如果你要采取第一种方法，那么请确保在 `report()` 方法中调用 `parent::report($exception)` ，这样父方法就会保留检查是否应该报告异常的逻辑，处理程序调用自定义异常的 `report()` ，并且在日志文件中记录异常。

> 如果你想要完全控制该如何报告异常，你可以停止调用 `parent::report($exception)` 并执行自己的操作，但请确保你已经查看父方法中的逻辑及其工作原理。

如果在自定义异常中实现了 `report()` 方法，那必须要注意，这个异常不会被记录在日志文件中。因为一旦异常处理程序在异常类中找到了一个 `report()` 方法，就会跳过这个步骤。