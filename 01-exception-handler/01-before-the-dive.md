#  Before The Dive

Laravel is shipped with two Kernels (cores), one for handling HTTP requests and the other for handling console commands, each Kernel instance has a `handle()` method that receives the input, either an HTTP Request or a Console Input, handles it, and returns a response. The entire handling process is wrapped inside a try...catch:

```php
try {
    $response = $this->processTheInput($input);
} catch (Exception $e) {
    $this->reportException($e);
    // This calls the report() method of `App\Exceptions\Handler`

    $response = $this->renderException($request, $e);
    // This calls the render() method of `App\Exceptions\Handler`
}

return $response;
```

As you can see all exceptions that aren't handled from within your application are caught at this point and then laravel performs two tasks:

1. It reports the exception to different channels as configured.
2. It converts the exception instance to a displayable format and renders the output to the end user.

>The two Kernels are accessed from "app\Http\Kernel.php" for the HTTP Kernel, and "app\Console\Kernel.php" for the console Kernel. Both classes extend their respective base Kernel Illuminate class, it's very interesting to take a deeper look on how things work there.

## So that's how all exceptions are handled?

There are some situations where exceptions are internally handled, for example inside a daemon queue worker to prevent the script from exiting when a worker encounters an exception while pulling new jobs or executing them Laravel catches exceptions and only reports them but never renders:

```php
try {
    // Get new jobs or execute a job
} catch (Exception $e) {
    $this->exceptions->report($e);
}
```

Another interesting behaviour is while handling an exception that's returned during running routes or middleware, before the exception reaches the try..catch statement inside the HTTP Kernel it's actually reported and converted to a response object inside the pipeline, that way exceptions won't stop the script from running all assigned after middleware.

## Why would I want that?

For example if you have a middleware that logs all requests & responses passing by your application you'll usually want to register it as an "After Middleware":

```php
class LoggerMiddleware
{
    public function handle($request, Closure $next)
    {
        $response = $next($request);

        // Send the request and the response to your logging server.

        return $response;
    }
}
```

So here any exceptions that occurs in any layer of your application won't stop Laravel from running this after middleware so that you can collect information about the actual response that your user received.

You can find information about the actual exception that was thrown by looking into the exception property of the response object:

```php
if ($response->exception){
    // ...
}
```

## In case of any other un-handled exceptions:

While bootstrapping the application Laravel sets PHP's error reporting to report all errors, it also assign custom handlers to errors and exceptions:

```php
// Turn on error reporting
error_reporting(-1);

// Register a custom error handler
set_error_handler([$this, 'handleError']);

// Register a custom exception handler
set_exception_handler([$this, 'handleException']);

// Handle when the script is done
register_shutdown_function([$this, 'handleShutdown']);
```

Inside those custom handlers Laravel tries to format any un-handled exceptions, report them, and present a displayable response to the end user.
