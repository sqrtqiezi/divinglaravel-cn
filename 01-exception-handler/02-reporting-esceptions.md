# Reporting Exceptions

Being able to track exceptions encountered inside your application is useful to understand what went wrong, without any configuration from your part all exceptions are logged inside a `storage/logs/laravel.log` file with full stack trace, so at any point of time you can go take a look at that file and see if anything wrong happened while your application is running.

## But sometimes I don't care if certain exceptions are thrown

Laravel is taking that into account, and before reporting any exception it checks if you really want it to be reported, actually by default laravel doesn't report certain exception types:

* Authentication and Authorization exceptions
* Model Not Found exceptions
* Token mismatch exceptions
* Validation exceptions
* HTTP exceptions
* Response exceptions

You can also define your own set of exceptions that you don't want to report from within the `App\Exceptions\Handler` class, locate the `$dontReport` property and provide the list as an array:

```php
class Handler extends ExceptionHandler
{
    protected $dontReport = [
        MyCustomException::class
    ];
}
```

## Built-in error reporting

Laravel is shipped with 4 different error modes, these modes are built on top of the [Monolog](https://github.com/Seldaek/monolog) library:

1. Single log file: Logs exceptions into a `storage/logs/laravel.log` file.
2. Daily log files: Logs exceptions by creating daily log files inside `storage/logs`
3. Syslog handler: Logs the exception into the syslog. [know more about that](https://community.rapid7.com/community/insightops/blog/2017/05/23/what-is-syslog)
4. Error log handler: Logs the exception into PHP's error log.

By default the Single File mode is selected, but you can switch to any of the modes by changing the `config/app.php["log"]` configuration, however if you're going to use the daily-Log mode laravel will delete older log files and only keep the latest 5 days, you can configure how many days the logs should be kept by changing the `log_max_files` configuration.

You can also use a different Monolog handler by placing a call to the `configureMonologUsing` method in your `bootstrap/app.php` file before returning the `$app` instance:

```php
$app->configureMonologUsing(function ($monolog) {
    $monolog->pushHandler(
        new SlackWebhookHandler($webhookUrl)
    );
});

return $app;
```

>If you'd like to have a deeper look at how this is all configured check the Illuminate\Log\LogServiceProvider, this is where the logger is created and Monolog is configured.

## What if I want to report exceptions in a different way?

You can do that in two ways:

* From within the `report()` method of your `App\Exceptions\Handler` class you can integrate your own reporting mechanism, this method is called whenever an exception should be reported and it's a good place for that kind of custom reporting
* Or you can define a `report()` method in your custom exception and define your own reporting mechanism there.

If you're going to take the first approach then make sure you call `parent::report($exception)` inside the `report()` method, the parent method hold the logic for checking if the exception should be reported or not, handles calling the `report()` method of custom exceptions, and also logging the exception to the log file.

> If you want to take full control of how exceptions are reported, you can stop calling parent::report($exception) and do your own thing but make sure you take a look at the logic inside that parent method to see how it works.

In case you implement a `report()` method inside a custom exception you have to be aware that this exception won't be logged in the log files, the Exceptional handler skips this step once a `report()` method is found on the exception class.