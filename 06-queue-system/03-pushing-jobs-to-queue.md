# Pushing Jobs To Queue

There are several ways to push jobs into the queue:

```php
Queue::push(new InvoiceEmail($order));

Bus::dispatch(new InvoiceEmail($order));

dispatch(new InvoiceEmail($order));

(new InvoiceEmail($order))->dispatch();
```

As explained in a [previous dive](https://divinglaravel.com/queue-system/preparing-jobs-for-queue), calls on the `Queue` facade are calls on the queue driver your app uses, calling the `push` method for example is a call to the `push` method of the `Queue\DatabaseQueue` class in case you're using the database queue driver.

There are several useful methods you can use:

```php
// Push the job on a specific queue
Queue::pushOn('emails', new InvoiceEmail($order));

// Push the job after a given number of seconds
Queue::later(60, new InvoiceEmail($order));

// Push the job on a specific queue after a delay
Queue::laterOn('emails', 60, new InvoiceEmail($order));

// Push multiple jobs
Queue::bulk([
    new InvoiceEmail($order),
    new ThankYouEmail($order)
]);

// Push multiple jobs on a specific queue
Queue::bulk([
    new InvoiceEmail($order),
    new ThankYouEmail($order)
], null, 'emails');
```

After calling any of these methods, the selected queue driver will store the given information in a storage space for workers to pick up on demand.

## Using the command bus

Dispatching jobs to queue using the command bus gives you extra control; you can set the selected `connection`, `queue`, and `delay` from within your job class, decide if the command should be queued or run instantly, send the job through a pipeline before running it, actually you can even handle the whole queueing process from within your job class.

The `Bus` facade proxies to the `Contracts\Bus\Dispatcher` container alias, this alias is resolved into an instance of `Bus\Dispatcher` inside `Bus\BusServiceProvider`:

```php
$this->app->singleton(Dispatcher::class, function ($app) {
    return new Dispatcher($app, function ($connection = null) use ($app) {
        return $app[QueueFactoryContract::class]->connection($connection);
    });
});
```

So doing `Bus::dispatch()` calls the `dispatch()` method of the `Bus\Dispatcher` class:

```php
public function dispatch($command)
{
    if ($this->queueResolver && $this->commandShouldBeQueued($command)) {
        return $this->dispatchToQueue($command);
    } else {
        return $this->dispatchNow($command);
    }
}
```

This method decides if the given job should be dispatched to queue or run instantly, the `commandShouldBeQueued()` method checks if the job class is an instance of `Contracts\Queue\ShouldQueue`, so using this method your job will only be queued in case you implement the `ShouldQueue` interface.

We're not going to look into `dispatchNow` in this dive, it'll be discussed in detail when we dive into how workers run jobs. For now let's look into how the Bus dispatches your job to queue:

```php
public function dispatchToQueue($command)
{
    $connection = isset($command->connection) ? $command->connection : null;

    $queue = call_user_func($this->queueResolver, $connection);

    if (! $queue instanceof Queue) {
        throw new RuntimeException('Queue resolver did not return a Queue implementation.');
    }

    if (method_exists($command, 'queue')) {
        return $command->queue($queue, $command);
    } else {
        return $this->pushCommandToQueue($queue, $command);
    }
}
```

First Laravel checks if a `connection` property is defined in your job class, using the property you can set which connection Laravel should send the queued job to, if no property was defined `null` will be used and in such case Laravel will use the default connection.

Using the connection value, Laravel uses a queueResolver closure to build the instance of the queue driver that should be used, this closure is set inside `Bus\BusServiceProvider` while registering the Dispatcher instance:

```php
function ($connection = null) use ($app) {
    return $app[Contracts\Queue\Factory::class]->connection($connection);
}
```

`Contracts\Queue\Factory` is an alias for `Queue\QueueManager`, so in other words this closure returns an instance of QueueManager and sets the desired connection for the manager to know which driver to use.

Finally the `dispatchToQueue` method checks if the job class has a `queue` method, if that's the case the dispatcher will just call this method giving you full control over how the job should be queued, you can select the queue, assign delay, set maximum retries, timeout, etc...

In case no `queue` method was found, a call to `pushCommandToQueue()` calls the proper `push`method on the selected queue driver:

```php
protected function pushCommandToQueue($queue, $command)
{
    if (isset($command->queue, $command->delay)) {
        return $queue->laterOn($command->queue, $command->delay, $command);
    }

    if (isset($command->queue)) {
        return $queue->pushOn($command->queue, $command);
    }

    if (isset($command->delay)) {
        return $queue->later($command->delay, $command);
    }

    return $queue->push($command);
}
```

The dispatcher checks for `queue` and `delay` properties in your Job class & picks the appropriate queue method based on that.

#### So I can set the queue, delay, and connection inside the job class?

Yes, you can also set a `tries` and `timeout` properties and the queue driver will use these values as well, here's how your job class might look like:

```php
class SendInvoiceEmail{
    public $connection = 'default';

    public $queue = 'emails';

    public $delay = 60;

    public $tries = 3;

    public $timeout = 20;
}
```

## Setting job configuration on the fly

Using the `dispatch()` global helper you can do something like this:

```php
dispatch(new InvoiceEmail($order))
        ->onConnection('default')
        ->onQueue('emails')
        ->delay(60);
```

This only works if you use the `Bus\Queueable` trait in your job class, this trait contains several methods that you may use to set some properties on the job class before dispatching it, for example:

```php
public function onQueue($queue)
{
    $this->queue = $queue;

    return $this;
}
```

#### But in your example we call the methods on the return of dispatch()!

Here's the trick:

```php
function dispatch($job)
{
    return new PendingDispatch($job);
}
```

This is the definition of the `dispatch()` helper in `Foundation/helpers.php`, it returns an instance of `Bus\PendingDispatch` and inside this class we have methods like this:

```php
public function onQueue($queue)
{
    $this->job->onQueue($queue);

    return $this;
}
```

So when we do `dispatch(new JobClass())->onQueue('default')`, the `onQueue` method of `PendingDispatch` will call the `onQueue` method on the job class, as we mentioned earlier job classes need to use the `Queueable` trait for all this to work.

#### Then where's the part where the Dispatcher::dispatch method is called?

Once you do `dispatch(new JobClass())->onQueue('default')` you'll have the job instance ready for dispatching, the actual work happens inside `PendingDispatch::__destruct()`:

```php
public function __destruct()
{
    app(Dispatcher::class)->dispatch($this->job);
}
```

This method, when called, will resolve an instance of `Dispatcher` from the container and call the `dispatch()` method on it. A `__destruct()` is a PHP magic method that's called when all references to the object no longer exist or when the script terminates, and since we don't store a reference to the `PendingDispatch` instance anywhere the `__destruct` method will be called immediately:

```php
// Here the destructor will be called rightaway
dispatch(new JobClass())->onQueue('default');

// Here the desctructor will be called if we call unset($temporaryVariable)
// or when the script finishes execution.
$temporaryVariable = dispatch(new JobClass())->onQueue('default');
```

### Using the Dispatchable trait

You can use the `Bus\Dispatchable` trait on your job class to be able to dispatch your jobs like this:

```php
(new InvoiceEmail($order))->dispatch();
```

Here's how the dispatch method of `Dispatchable` looks like:

```php
public static function dispatch()
{
    return new PendingDispatch(new static(...func_get_args()));
}
```

As you can see it uses an instance of `PendingDispatch`, that means we can do something like:

```php
(new InvoiceEmail($order))->dispatch()->onQueue('emails');
```

