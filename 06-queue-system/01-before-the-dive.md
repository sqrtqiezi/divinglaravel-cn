# Before The Dive

Laravel receives a request, does some work, and then returns a response to the user, this is the normal synchronous workflow of a web server handling a request, but sometimes you need to do some work behind the scenes that doesn't interrupt or slowdown this flow, say for example sending an invoice email to the user after making an order, you don't want the user to wait until the mail server receives the request, builds the email message, and then dispatches it to the user, instead you'd want to show a "Thank You!" screen to the user so that he can continue with his life while the email is being prepared and sent in the background.

Laravel is shipped with a built-in queue system that helps you run tasks in the background & configure how the system should react in different situation using a very simple API.

You can manage your queue configurations in `config/queue.php`, by default it has a few connections that use different queue drivers, as you can see you can have several queue connections in your project and use multiple queue drivers too.

We'll be looking into the different configurations down the road, but let's first take a look at the API:

```php
Queue::push(new SendInvoice($order));

return redirect('thank-you');
```

The `Queue` facade here proxies to the `queue` container alias, if we take a look at `Queue\QueueServiceProvider` we can see how this alias is registered:

```php
protected function registerManager()
{
    $this->app->singleton('queue', function ($app) {
        return tap(new QueueManager($app), function ($manager) {
            $this->registerConnectors($manager);
        });
    });
}
```

So the `Queue` facade proxies to the `Queue\QueueManager` class that's registered as a singleton in the container, we also register the connectors to different queue drivers that Laravel supports using `registerConnectors()`:

```php
public function registerConnectors($manager)
{
    foreach (['Null', 'Sync', 'Database', 'Redis', 'Beanstalkd', 'Sqs'] as $connector) {
        $this->{"register{$connector}Connector"}($manager);
    }
}
```

This method simply calls the `register{DriverName}Connector` method, for example it registers a Redis connector:

```php
protected function registerRedisConnector($manager)
{
    $manager->addConnector('redis', function () {
        return new RedisConnector($this->app['redis']);
    });
}
```

> The `addConnector()` method stores the values to a `QueueManager::$connectors` class property.

A connector is just a class that contains a `connect()` method which creates an instance of the desired driver on demand, here's how the method looks like inside `Queue\Connectors\RedisConnector`:

```php
public function connect(array $config)
{
    return new RedisQueue(
        $this->redis, $config['queue'],
        Arr::get($config, 'connection', $this->connection),
        Arr::get($config, 'retry_after', 60)
    );
}
```

So now The QueueManager is registered into the container and it knows how to connect to the different built-in queue drivers, if we look at that class we'll find a `__call()` magic method at the end:

```php
public function __call($method, $parameters)
{
    return $this->connection()->$method(...$parameters);
}
```

All calls to methods that doesn't exist in the `QueueManager` class will be sent to the loaded driver, for example when we called the `Queue::push()` method, what happened is that the manager selected the desired queue driver, connected to it, and called the `push` method on that driver.

### How does the manager know which driver to pick?

Let's take a look at the `connection()` method:

```php
public function connection($name = null)
{
    $name = $name ?: $this->getDefaultDriver();

    if (! isset($this->connections[$name])) {
        $this->connections[$name] = $this->resolve($name);

        $this->connections[$name]->setContainer($this->app);
    }

    return $this->connections[$name];
}
```

When no connection name specified, Laravel will use the default queue connection as per the config files, the `getDefaultDriver()` returns the value of `config/queue.php['default']`:

```php
public function getDefaultDriver()
{
    return $this->app['config']['queue.default'];
}
```

Once a driver name is defined the manager will check if that driver was loaded before, only if it wasn't it starts to connect to that driver and load it using the `resolve()` method:

```php
protected function resolve($name)
{
    $config = $this->getConfig($name);

    return $this->getConnector($config['driver'])
                ->connect($config)
                ->setConnectionName($name);
}
```

First it loads the configuration of the selected connection from your `config/queue.php` file, then it locates the connector to the selected driver, calls the `connect()` method on it, and finally sets the connection name for further use.

### Now we know where to find the push method

Yes, when you do `Queue::push()` you're actually calling the `push` method on the queue driver you're using, each driver handles the different operations in its own way but Laravel provides you a unified interface that you can use to give the queue manager instructions despite of what driver you use.

### What if I want to use a driver that's not built-in?

Easy, you can call `Queue::addConnector()` with the name of your custom driver as well as a closure that explains how a connection to that driver could be acquired, also make sure that your connector implements the `Queue\Connectors\ConnectorInterface` interface.

Once you register the connector, you can use any connection that uses this driver:

```php
Queue::connection('my-connection')->push(...);
```
