#  写在前面

Laravel 配有两个内核，一个用于处理 HTTP 请求，另一个则用于处理控制台命令。每个内核的实例中都有一个 `handle()` 方法，用来接收表单输入的内容、HTTP 请求或者控制台输入的内容，然后对这些内容进行处理之后再返回响应。 这整个处理的过程被包装在一个 try … catch 中：

```php
try {
    $response = $this->processTheInput($input);
} catch (Exception $e) {
    $this->reportException($e);
    // 这里调用了 `App\Exceptions\Handler` 中的 report() 方法

    $response = $this->renderException($request, $e);
    // 这里调用了 `App\Exceptions\Handler` 中的 render() 方法
}

return $response;
```

正如你看到的，在应用程序中所有未处理的异常都在这里被捕获，接着 Laravel 会执行两个任务：

1. 根据配置向相应的 Kernel 汇报异常。
2. 将异常实例转换为可显示的格式，再输出呈现给最终用户。

>两个内核可以从 HTTP 内核的 “app\Http\Kernel.php” 和控制台内核的 “app\Console\Kernel.php” 访问。 这两个类都扩展了它们各自在 Illuminate 中的 Kernel 基类，深入了解它们是怎么工作是一件非常有趣的事情。

#### 那么所有的异常都是如何被处理的呢？

在某些情况下，异常会在内部处理。例如在队列的守护进程中，Laravel 捕获异常但仅做报告而不会呈现，以防止在拉取新作业或在执行它们的过程中遇到异常时脚本退出：

```php
try {
    // 获得新任务或执行任务
} catch (Exception $e) {
    $this->exceptions->report($e);
}
```

另一个有趣的行为是在处理运行中的路由或中间件期间返回的异常时，在异常到达 HTTP 的 Kernel内的 try..catch 语句之前，它实际上被报告并转换为管道中的响应对象，这样异常不会停止该脚本运行所有被分配的后置中间件。

#### 为什么要这样？

例如，如果你有一个中间件来记录通过应用程序的所有请求和响应，你通常会想要注册一个「后置中间件」：

```php
class LoggerMiddleware
{
    public function handle($request, Closure $next)
    {
        $response = $next($request);

        // 发送请求和响应到你的日志服务中

        return $response;
    }
}
```

因此，出现在应用程序中任何层的任何异常都不会停止 Laravel 运行后置中间件，这样你可以收集有关用户收到的实际响应的信息。

你可以通过查看响应对象的 **exception** 属性来查找关于抛出实际异常的信息：

```php
if ($response->exception){
    // ...
}
```

#### 如果有任何未处理的异常：

在引导应用程序时， Laravel 设置 PHP 的错误报告会报告所有错误，也会为自定义处理程序分配错误和异常：

```php
// 打开错误报告
error_reporting(-1);

// 注册自定义错误处理程序
set_error_handler([$this, 'handleError']);

// 注册自定义异常处理程序
set_exception_handler([$this, 'handleException']);

// 当脚本完成时处理
register_shutdown_function([$this, 'handleShutdown']);
```

在这些自定义处理程序中，Laravel 尝试格式化任何未处理的异常并将其报告，再给最终的用户呈现可显示的响应。
