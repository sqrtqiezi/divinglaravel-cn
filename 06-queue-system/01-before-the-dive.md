# 写在前面

Laravel 接收请求，做些工作，然后返回响应给用户，这是 Web 服务器处理一个请求的的正常同步工作流程。但有时需要在幕后做一些不会中断或减缓此流程的工作。例如，在用户下订单之后向其发送发票的电子邮件，你不会想要让用户等到邮件服务器收到请求，组装邮件信息，然后将其发给用户。而是在用户的显示屏中显示「谢谢」，以便后台在进行电子邮件的发送的同时他可以继续他的生活。

Laravel 配有内置的队列系统，能帮助你在后台运行任务，并配置系统如何在不同情况下使用简单的 API 进行响应。

可以在 `config/queue.php` 中管理队列的配置。默认情况下，就你可以看到的，它有一些使用不同的队列驱动程序的连接。你的项目中可以有多个队列连接，也可以有多个队列驱动程序。

我们将研究不同的配置，但请先看看 API：

```php
Queue::push(new SendInvoice($order));

return redirect('thank-you');
```

`Queue` 门面（facade）中有 `queue` 容器别名作为代理，我们可以在 `Queue\QueueServiceProvider` 中看到这个别名是如何注册的：

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

上面的代码中， `Queue` 门面是代理在容器中注册为单例（singleton）的  `Queue\QueueManager` 类。另外还将连接器注册到不同队列驱动程序中，Laravel 也支持使用 `registerConnectors()` ：

```php
public function registerConnectors($manager)
{
    foreach (['Null', 'Sync', 'Database', 'Redis', 'Beanstalkd', 'Sqs'] as $connector) {
        $this->{"register{$connector}Connector"}($manager);
    }
}
```

这个方法只需调用 `register{DriverName}Connector` 方法，例如注册一个 Redis 连接器：

```php
protected function registerRedisConnector($manager)
{
    $manager->addConnector('redis', function () {
        return new RedisConnector($this->app['redis']);
    });
}
```

> `addConnector()` 方法将值存储到一个 `QueueManager::$connectors` 类的属性。

连接器只是一个类，它包含一个 `connect()` 方法，它根据需要创建所需驱动程序的实例，下面是 `Queue\Connectors\RedisConnector` 内部的方法：

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

所以现在 QueueManager 被注册到容器中，它知道如何连接到不同的内置队列驱动程序。而 QueueManager 这个类的最后，有一个魔术方法 `__call()`：

```php
public function __call($method, $parameters)
{
    return $this->connection()->$method(...$parameters);
}
```

在 `QueueManager` 类中所有不存在的方法的调用都会被发送到加载的驱动程序，例如当我们调用 `Queue::push()` 方法时，发生的是管理器选择了所需的队列驱动程序，连接，并在该驱动程序上调用 `push` 方法。

#### 管理器如何知道要选择哪个驱动？

我们来看一下 `connection()` 方法：

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

当没有指定连接名称时，Laravel 将根据配置文件使用默认队列连接， `getDefaultDriver()` 方法会返回 `config/queue.php['default']` 上的值：

```php
public function getDefaultDriver()
{
    return $this->app['config']['queue.default'];
}
```

一旦定义了驱动程序名称，管理器将检查该驱动程序是否已加载，只有当它不是开始连接到该驱动程序并使用 `resolve()` 方法时加载它：

```php
protected function resolve($name)
{
    $config = $this->getConfig($name);

    return $this->getConnector($config['driver'])
                ->connect($config)
                ->setConnectionName($name);
}
```

首先从 `config/queue.php` 文件加载所选连接的配置，然后将连接器定位到所选驱动程序，调用其上的 `connect()` 方法，最后设置连接名称以供进一步使用。

#### 现在我们知道在哪里可以找到 push 方法

当你执行 `Queue::push()` 时，你实际上正在调用你正在使用的队列驱动程序的 `push`  方法，每个驱动程序都以自己的方式处理不同的操作，但是 Laravel 提供了一个统一的界面，这样不管使用了什么驱动程序，都可以使用它来给队列管理器配置指令。

#### 如果想使用不是内置的驱动程序怎么办？

很简单，你可以使用自定义驱动程序的名称然后调用 `Queue::addConnector()` 以及一个闭包，用来说明如何获取与该驱动程序的连接，当然，你还要确保你的连接器实现了 `Queue\Connectors\ConnectorInterface` 接口。

一旦注册连接器后，就可以使用任何使用此驱动程序的连接：

```php
Queue::connection('my-connection')->push(...);
```
