# Laravel 深入浅出之队列篇（一）

Laravel 接收到请求，处理完之后马上返回响应给用户。Web 服务器正常处理请求的同步工作流程。但有时候，并不一定需要等到某些事情处理完才能返回响应给用户。例如，用户下订单之后要向其发送发票的电子邮件，你不会想要让用户等到服务器确认邮件已经发送到他邮箱之后才在用户的界面上显示「谢谢」。

Laravel 自身配有队列系统来实现上述需求，只需使用非常简单的 API 就能配置系统在不同情况下的响应方式。

队列的配置文件放在 `config/queue.php` 中，该文件有各种不同队列的驱动程序的连接。这意味着一个项目里面可以有多个队列，也可以有多个队列驱动。

配置的东西晚点再研究，现在先来看看它的 API：

```php
Queue::push(new SendInvoice($order));

return redirect('thank-you');
```

`Queue` 门面（facade）中有 `queue` 容器别名作为代理，我们可以在 `Queue\QueueServiceProvider` 中看到这个别名是如何被注册的：

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

可以看到，`Queue` 门面代理容器中依赖注入的 `Queue\QueueManager` 类，还使用了 `registerConnectors()` 来注册 Laravel 支持的不同队列驱动程序：

```php
public function registerConnectors($manager)
{
    foreach (['Null', 'Sync', 'Database', 'Redis', 'Beanstalkd', 'Sqs'] as $connector) {
        $this->{"register{$connector}Connector"}($manager);
    }
}
```

这个方法只是调用了 `register{DriverName}Connector` 方法，例如我们要注册一个 `Redis` 的连接器：

```php
protected function registerRedisConnector($manager)
{
    $manager->addConnector('redis', function () {
        return new RedisConnector($this->app['redis']);
    });
}
```

> `addConnector()` 方法将值存储到 `QueueManager::$connectors` 类的属性中。

这些连接器类只包含了 `connect()` 方法，该方法根据需要创建所需驱动程序的实例，以下是  `Queue\Connectors\RedisConnector` 的 `connect()` 方法：

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

也就是说，当 QueueManager 被注册到容器中之后，它知道如何连接到不同的队列驱动程序。而 QueueManager 这个类的最后有一个魔术方法 `__call()`：

```php
public function __call($method, $parameters)
{
    return $this->connection()->$method(...$parameters);
}
```

对 `QueueManager` 类中不存在的方法的所有调用都会被发送到加载的驱动程序，例如当我们调用 `Queue::push()` 方法时，QueueManager 调用了连接队列的驱动程序，并在该驱动程序上调用 `push` 方法。

#### QueueManager 是如何知道要选择哪个驱动的？

让我们来看一下 `connection()` 方法：

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

如果没有指定连接名称，Laravel 会根据配置文件使用默认队列连接， `getDefaultDriver()` 方法会返回 `config/queue.php['default']` 上的值：

```php
public function getDefaultDriver()
{
    return $this->app['config']['queue.default'];
}
```

一旦定义了驱动程序名称，QueueManager 会检查该驱动程序是否已加载，当它连接到该驱动程序时，便使用 `resolve()` 方法时加载它：

```php
protected function resolve($name)
{
    $config = $this->getConfig($name);

    return $this->getConnector($config['driver'])
                ->connect($config)
                ->setConnectionName($name);
}
```

我们可以来看看 QueueManager 类中的 `resolve` 方法。首先，它会从 `config/queue.php` 文件加载你选择的连接的配置，然后将连接器定位到所选驱动程序，调用该连接器的 `connect()` 方法，最后设置连接名称以供进一步使用。

#### 在哪里可以找到 push 方法

当你执行 `Queue::push()` 时，你实际上正在调用你正在使用的队列驱动程序的 `push`  方法，每个驱动程序都以自己的方式处理不同的操作。而 Laravel 提供了一个统一的界面，这样不管你使用了什么驱动程序，都可以使用它来给 QueueManager 下发指令。

#### 如果想使用不是内置的驱动程序怎么办？

很简单，你可以使用自定义驱动程序的名称然后调用 `Queue::addConnector()` 和一个用来说明如何获取与该驱动程序的连接的闭包，当然，你还要确保你的连接器实现了 `Queue\Connectors\ConnectorInterface` 接口。

注册好连接器之后就可以使用任何连接来使用这个驱动：

```php
Queue::connection('my-connection')->push(...);
```
