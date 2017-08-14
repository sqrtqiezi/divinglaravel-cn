# 准备队列任务

推送到队列的每个任务都存储在按执行顺序排序的某些存储空间中，这个存储位置可以是 MySQL 数据库、Redis 存储或者像 Amazon SQS 这样的第三方服务。

在这次深入探究之后，我们将探索开发者如何抓取这些任务并开始执行，但在此之前，先看看是如何存储任务。下面是为每个存储任务所保留的属性：

- 应该运行什么队列
- 此任务的尝试次数（最初为零）
- 任务由开发者保留的时间
- 任务时间可供开发者保留
- 创建任务的时间
- 任务的有效载荷

#### 队列是什么？

应用程序发送几种类型的任务到队列。默认情况下，所有任务都放在单个队列中。然而，你可能希望将邮件任务放在不同的队列中，并指示专门的程序仅在此队列运行任务，这确保其他任务不会延迟发送电子邮件，因为它有自己的专用队列。

### 尝试次数

默认情况下，如果运行失败，队列管理器将继续运行这个任务。除非设置了最大失败次数，否则将一直重复运行这个任务，当任务运行失败达到该数量时，该任务将会真正被标记为失败，而程序也不会再继续运行它。

### 保留时间

Once a worker picks a job we mark it as reserved and store the timestamp of when that happened, next time a worker tries to pick a new job it won't pick a reserved one unless the reservation lock is expired, but more on that later.

一旦开发者选择了任务，我们将其标记为预约，并存储运行的时间戳。下一次队列尝试选择新的工作时，除非保留锁定已过期，否则不会选择保留位。

### 可用时间

By default a job is available once it's pushed to queue and workers can pick it right away, but sometimes you might need to delay running a job for sometime, you can do that by providing a delay while pushing the job to queue using the `later()` method instead of `push()`:

默认情况下，一旦任务被推送到队列，队列可以立即执行它。但有时可能需要延迟一段时间的工作，你可以通过在使用 `later()` 方法而不是 `push()` 将任务推送到队列时提供延迟：

```php
Queue::later(60, new SendInvoice())
```

`later()` 方法使用 `availableAt()` 方法来确定可用性时间：

```Php
protected function availableAt($delay = 0)
{
    return $delay instanceof DateTimeInterface
                        ? $delay->getTimestamp()
                        : Carbon::now()->addSeconds($delay)->getTimestamp();
}
```

可以传递一个 `DateTimeInterface` 的实例来设置确切的时间，或者也可以传递秒数，Laravel 会计算出可用时间。

## 有效载荷

The `push()` & `later()` methods use the `createPayload()` method internally to create the information needed to execute the job when it's picked by a worker. You can pass a job to queue in two formats:

当工作人员选择它时，`push()` 和 `later()` 方法在内部使用 `createPayload()` 方法来创建执行任务所需的信息。 可以通过两种格式将任务传递给队列：

```php
// Pass an object
Queue::push(new SendInvoice($order));

// Pass a string
Queue::push('App\Jobs\SendInvoice@handle', ['order_id' => $orderId])
```

In the second example while the worker is picking the job, Laravel will use the container to create an instance of the given class, this gives you the chance to require any dependencies in the job's constructor.

在第二个例子中，当开发者正在选择任务时，Laravel 将使用该容器来创建给定类的实例，这样可以有机会在任务的构造函数中要求任何依赖关系。

### 创建字符串任务的有效负荷

`createPayload()` calls `createPayloadArray()` internally which calls the `createStringPayload()` method in case the job type is non-object:

`createPayload()` 在内部调用 `createPayloadArray()`，调用 `createStringPayload()` 方法，以防任务类型为非对象：

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

The `displayName` of a job is a string you can use to identify the job that's running, in case of non-object job definitions we use the the job class name as the `displayName`.

任务的 `displayName` 是可用于标识正在运行的任务的字符串，在非对象任务定义的情况下，我们使用任务名称作为 `displayName`。

Notice also that we store the given data in the job payload.

还要注意，我们将给定的数据存储在任务有效载荷中。

### 创建对象任务的有效负载

Here's how an object-based job payload is created:

以下是创建基于对象的任务有效负载的方式：

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

Since we already have the instance of the job we can extract some useful information from it, for example the `getDisplayName()` method looks for a `displayName()` method inside the job instance and if found it uses the return value as the job name, that means you can add such method in your job class to be in control of the name of your job in queue.

由于我们已经有了这个任务的实例，我们可以从中提取一些有用的信息。例如，  `getDisplayName()` 方法在任务实例中查找一个 `displayName()` 方法，如果发现它使用返回值作为任务名称，那意味着你可以在任务类中添加这样的方法来控制队列中任务的名称。

```Php
protected function getDisplayName($job)
{
    return method_exists($job, 'displayName')
                    ? $job->displayName() : get_class($job);
}
```

We can also extract the value of the maximum number a job should be retried and the timeout for the job, if you pass these values as class properties Laravel stores this data into the payload for use by the workers later.

还可以提取应该重试的任务的最大数量的值和作业的超时值，如果你将这些值作为类属性传递，Laravel 将此数据存储到有效载荷中，以供以后使用。

As for the `data` attribute, Laravel stores the class name of the job as well as a serialized version of that job.

对于 `data` 属性，Laravel 存储任务的类名称以及该任务的序列化版本。

#### 那么如何传递我自己的数据

In case you chose to pass an object-based job you can't provide a data array, you can store any data you need inside the job instance and it'll be available once un-serialized.

如果你选择传递基于对象的任务，则无法提供数据数组，你可以在任务实例中存储所需的任何数据，并且一旦未序列化，它将可用。

#### 为什么我们通过不同的类作为「任务」参数

`Queue\CallQueuedHandler@call` is a special class Laravel uses while running object-based jobs, we'll look into it in a later stage.

`Queue\CallQueuedHandler@call` 是 Laravel 在运行基于对象的任务时使用的一个特殊类，我们将在稍后的阶段进行研究。

## 序列化作业

Serializing a PHP object generates a string that holds information about the class the object is an instance of as well as the state of that object, this string can be used later to re-create the instance.

序列化 PHP 对象会生成一个字符串，该字符串保存有关该对象是该实例的类的信息以及该对象的状态，此字符串以后可用于重新创建该实例。

In our case we serialize the job object in order to be able to easily store it somewhere until a worker is ready to pick it up & run it, while creating the payload for the job we serialize a clone of the job object:

在我们的情况下，我们序列化任务对象，以便能够轻松地将其存储在某个位置，直到工作人员准备好接收并运行它，同时为任务创建有效负载，我们将序列化任务对象的克隆：

```php
serialize(clone $job);
```

#### 但为什么要克隆？ 为什么不用序列化对象本身？

While serializing the job we might need to do some transformation to some of the job properties or properties of any of the instances our job might be using, if we pass the job instance itself transformations will be applied to the original instances while this might not be desired, let's take a look at an example:

在序列化任务时，我们可能需要对我们的任务可能使用的任何实例的某些任务属性或属性进行一些转换，如果我们传递任务实例本身，转换将应用于原始实例，而这可能不是期望的，让我们来看一个例子：

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

While creating the payload for the `SendInvoice` job we're going to serialize that instance and all its child objects, the Invoice object, but PHP doesn't always work well with serializing files and the Invoice object has a property called `$pdf` which holds a file, for that we might want to store that file somewhere, keep a reference to it in our instance while serializing and then when we un-serialize we bring back the file using that reference.

在为 `SendInvoice` 任务创建有效内容时，我们将序列化该实例及其所有子对象（Invoice对象），但PHP并不总是适用于序列化文件，并且Invoice对象具有称为  `$pdf`  的属性，该属性包含 文件，因为我们可能想要将该文件存储在某个地方，在序列化期间保留对我们实例的引用，然后当我们进行非序列化时，我们使用该引用返回文件。

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

The `__sleep()` method is automatically called before PHP starts serializing our object, in our example we transform our pdf property to hold the path to the PDF resource instead of the resource itself, and inside `__wakup()` we convert that path back to the original value, that way we can safely serialize the object.

在PHP开始序列化我们的对象之前，`__sleep()` 方法被自动调用，在我们的示例中，我们将我们的pdf属性转换为保存PDF资源的路径而不是资源本身，而在 `__wakup()` 中，我们将该路径转换回原始 值，这样我们可以安全地序列化对象。

Now if we dispatch our job to queue we know that it's going to be serialized under the hood:

现在如果我们派遣我们的工作排队，我们知道它将被序列化：

```php
$invoice = new Invoice($pdf, 'Customer #123');

Queue::push(new SendInvoice($invoice));

dd($invoice->pdf);
```

However, if we try to look at the invoice pdf property after sending the job to queue we'll find that it holds the path to the resource, it doesn't hold the resource anymore, that's because while serializing the Job PHP serialized the original invoice instance as well since it was passed by reference to the SendInvoice instance.

然而，如果我们在发送作业到队列后尝试查看发票pdf属性，我们会发现它拥有资源的路径，它不再占用资源，这是因为序列化作业PHP序列化原始 发票实例，因为它通过引用传递给 SendInvoice 实例。

Here's when cloning becomes handy, while cloning the `SendInvoice` object PHP will create a new instance of that object but that new instance will still hold reference to the original Invoice instance, but we can change that:

以下是克隆变得方便时，克隆 `SendInvoice` 对象时 PHP 将创建该对象的新实例，但新的实例仍将保留对原始 Invoice（发票）实例的引用，但是我们可以更改：

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

Here we instruct PHP that whenever it clones an instance of the `SendInvoice` object it should use a clone of the invoice property in the new instance not the original one, that way the original Invoice object will not be affected while we serialize.

这里我们指示PHP，只要克隆 SendInvoice 对象的一个实例，它应该使用新实例中的 invoice 属性的克隆，而不是原始实例，那么原始 Invoice 对象在序列化时不会受到影响。