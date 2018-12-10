# 准备队列任务

我们推送到队列的每个任务都存储在按执行顺序排序的某个存储空间中，这个存储位置可以是 MySQL 数据库、Redis 存储或者像 Amazon SQS 这样的第三方服务。

接下来的深入浅出，我们来探索 worker 如何获取这些任务并开始执行它们。但在此之前，让我们先了解下任务是如何被存储的。下面是每个存储任务都应该具备的属性：

- 应该运行什么队列
- 失败任务的重试次数（最初为零）
- 失败任务重试的保留时间
- 任务的可执行时间
- 创建任务的时间
- 任务的有效负载

#### 队列是什么？

应用程序会将多种类型的任务发送到队列。默认情况下，所有任务都放在单个队列中。然而，你可能希望将邮件任务放在不同的队列中，并为此队列指派专属 worker 来运行任务，这确保不会因为其他任务而延迟发送电子邮件，因为它有自己专用的 worker。

### 重试次数

默认情况下，如果任务运行失败，队列管理器会继续运行这个任务。除非你设置了最大重试次数，否则它会一直重复运行这个任务。当任务重试达到该数字时，这个任务才会真正地标记为失败，而 worker 也不会再继续去尝试完成这个任务。

### 保留时间

我们标记为保留并存储其运行的时间戳的任务，除非保留锁定的时间已过期，否则就算 woker 尝试选择新任务，也不会选择保留区的任务。

### 可用时间

默认情况下，一旦任务被推送到队列，worker 就会立即执行它。但也有时候需要任务能被延迟执行，此时你可以通过使用 `later()` 方法而不是 `push()` 来推送延迟任务到队列：

```php
Queue::later(60, new SendInvoice())
```

`later()` 方法使用了 `availableAt()` 方法来确定可用时间：

```Php
protected function availableAt($delay = 0)
{
    return $delay instanceof DateTimeInterface
                        ? $delay->getTimestamp()
                        : Carbon::now()->addSeconds($delay)->getTimestamp();
}
```

如你所见，你可以传递一个 `DateTimeInterface` 的实例来设置确切的时间，或者也可以传递秒数，Laravel 会计算出可用时间。

## 有效负载

`push()` 和 `later()` 方法都使用了 `createPayload()` 方法来创建在 worker 选择执行任务时所需的信息。你可以用以下两种格式将任务传递给队列：

```php
// 传递对象
Queue::push(new SendInvoice($order));

// 传递字符串
Queue::push('App\Jobs\SendInvoice@handle', ['order_id' => $orderId])
```

在第二个例子中，Laravel 会使用容器来创建给定类的实例，这样就可以在任务的构造函数中使用依赖注入。

### 创建字符串任务的有效负载

`createPayload()` 调用 `createPayloadArray()`，如果任务类型是非对象，则调用 `createStringPayload()` 方法：

```php
protected function createStringPayload($job, $data)
{
    return [
        'displayName' => is_string($job) ? explode('@', $job)[0] : null,
        'job' => $job, 'maxTries' => null,
        'timeout' => null, 'data' => $data,
    ];
}
```

任务的 `displayName` 是个字符串，用于标识正在运行的任务，在非对象任务定义的情况下，我们使用任务名称作为 `displayName`。

还要注意，我们将给定的数据存储在任务有效负载中。

### 创建对象任务的有效负载

以下是如何创建基于对象的任务有效负载：

```Php
protected function createObjectPayload($job)
{
    return [
        'displayName' => $this->getDisplayName($job),
        'job' => 'Illuminate\Queue\CallQueuedHandler@call',
        'maxTries' => isset($job->tries) ? $job->tries : null,
        'timeout' => isset($job->timeout) ? $job->timeout : null,
        'data' => [
            'commandName' => get_class($job),
            'command' => serialize(clone $job),
        ],
    ];
}
```

既然我们已经有了这个任务的实例，那我们可以从中提取一些有用的信息。例如，`getDisplayName()` 方法在任务实例中查找 `displayName()` 方法，如果找到，则使用返回值作为任务名称。也就是你可以在任务类中添加这样的方法来控制队列中任务的名称。

```Php
protected function getDisplayName($job)
{
    return method_exists($job, 'displayName')
                    ? $job->displayName() : get_class($job);
}
```

我们还可以提取重试次数以及任务的超时时间，如果将这些值作为类的属性传递，Laravel 会将此数据存储到有效负载中，以供日后使用。

`data` 属性中存储了任务的类名以及该任务的序列化版本。

#### 如何传递数据

如果你选择传递基于对象的任务就无法提供数据数组，但你可以在任务实例中存储所需的数据，解序列化之后再来使用。

#### 为什么要将另一个类赋值给「 job」参数传递

`Queue\CallQueuedHandler@call` 是 Laravel 在运行基于对象的任务时使用的一个特殊类，我们将在稍后的阶段进行研究。

## 序列化任务

PHP 序列化对象时会生成一个字符串，该字符串包含有关该对象所属的类的信息以及该对象的状态，因此可以用这个字符串重新创建实例。

在我们的例子中，我们序列化任务对象，以便能够轻松地将其存储在某个地方，直到 worker 准备好接收并运行它。同时为了创建任务有效负载，我们序列化任务对象的克隆：

```php
serialize(clone $job);
```

#### 但为什么要克隆？ 而不序列化对象本身？

在序列化任务时，我们可能需要对某些任务属性或者实例的属性进行一些转换，如果我们传递任务实例本身，那便只会转化实例本身，而这显然是不行的，让我们看一个例子：

```php
class SendInvoice
{
    public function __construct(Invoice $invoice)
    {
        $this->invoice = $invoice;
    }
}

class Invoice
{
    public function __construct($pdf, $customer)
    {
        $this->pdf = $pdf;
        $this->customer = $customer;
    }
}
```

在为 `SendInvoice` 任务创建有效负载时，我们将序列化该实例及其所有子对象，即 Invoice 对象，但 PHP 并不总是能够很好地序列化文件，Invoice 对象具有称为  `$pdf`  的属性，该属性包含文件。为此我们要将该文件存储在某个地方，以便在序列化时能在实例中保留对它的引用，然后当我们进行解序列化时，我们就可以使用该引用返回的文件。

```php
class Invoice
{
    public function __construct($pdf, $customer)
    {
        $this->pdf = $pdf;
        $this->customer = $customer;
    }

    public function __sleep()
    {
        $this->pdf = stream_get_meta_data($this->pdf)['uri'];

        return ['customer', 'pdf'];
    }

    public function __wakeup()
    {
        $this->pdf = fopen($this->pdf, 'a');
    }
}
```

在 PHP 开始序列化我们的对象之前，`__sleep()` 方法会被自动调用，在我们的示例中，我们将 pdf 属性转换为保存 PDF 资源的路径而不是资源本身，而在 `__wakup()` 中，我们将该路径转换回源文件，这样我们才可以安全地序列化对象。

现在，如果我们将任务分配到队列，它会在后台进行序列化：

```php
$invoice = new Invoice($pdf, 'Customer #123');

Queue::push(new SendInvoice($invoice));

dd($invoice->pdf);
```

但是，如果我们在发送任务到队列后，尝试查看发票的 pdf 属性，我们会发现它保存了资源的路径，不再保留资源，这是因为在序列化任务时，PHP 序列化了原始发票实例，因为它是通过引用 SendInvoice 实例传递的。

Here's when cloning becomes handy, while cloning the `SendInvoice` object PHP will create a new instance of that object but that new instance will still hold reference to the original Invoice instance, but we can change that:

在这里如果使用克隆就变得很方便了。PHP 克隆 `SendInvoice` 对象时会创建该对象的新实例，新实例仍然保留对原始 Invoice 实例的引用，我们可以更改成这样：

```php
class SendInvoice
{
    public function __construct(Invoice $invoice)
    {
        $this->invoice = $invoice;
    }

    public function __clone()
    {
        $this->invoice = clone $this->invoice;
    }
}
```

这里我们告诉 PHP，克隆 SendInvoice 对象的实例时，它应该在使用新实例中使用克隆的 invoice 属性，而不是原始实例。这样我们序列化原始的 Invoice 对象时就不会受到影响。