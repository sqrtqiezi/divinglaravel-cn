#写在前面

Laravel Horizon 是一个队列管理器，让你能完全控制你的队列，它提供了一个漂亮的仪表板来配置任务处理方式、生成分析以及执行不同队列相关任务的方法。

在这次深入浅出中，我们将了解 Horizon 如何启动并使用不同的 worker 处理执行任务，以及如何收集有用的指标来全面了解应用程序如何分配和运行任务。

这一切都是从运行 `php artisan horizon` 开始的，这个命令扫描你的 `horizon.php` 配置文件，并根据以下配置启动队列的 worker：

```php
'production' => [
    'supervisor-1' => [
        connection => "redis",
        queue=> "notifications,emails",
        maxProcesses=> 5,
        minProcesses=> 1,
        delay=> 0,
        memory=> 128,
        timeout=> 60,
        sleep=> 3,
        maxTries=> 0,
        balance=> "simple", // 可以是 simple、auto 或者 null
        force=> false,
    ],
    'supervisor-2' => [
        // ...
    ]
],
```


在这里，你可以为每个环境定义任意数量的「监督者」（ supervisor ）。监督者是用来监控队列中的 worker 的系统进程 并确保它们能按照配置去工作，每个监督者可以设置以下内容：

1. 默认情况下，监督者监控你「默认」的那个队列。当然你也可以添加任意数量的队列。
2. 允许运行的最小和最多的 worker 的数量。
3. 失败任务延迟执行的时间。
4. worker 的内存限制。
5. worker 的超时时间。
6. 没有任务可执行时睡眠的时间。
7. 任务执行失败之后的重试次数。
8. 均衡策略。
9. worker 是否在维护模式下运行。

#### 什么是均衡策略？

你有下面三个选项：

1. 没有均衡
2. 简单均衡
3. 自动均衡

如果你不想要均衡负载，那监督者会让一个 worker 来同时处理所有队列中的任务。

但是，如果你选择 `simple` 作为均衡策略，Horizon 会为每个队列创建一个进程池。也就是说，如果你的监督者要处理 `emails` 和 `notifications` 两个队列，而你又将进程数设置为 4，那 Horizon 会创建有两个进程池，每个进程池分别监控两个 worker。

如果你希望 Horizon 根据队列的繁忙程度来分配负载，就选择 `auto` 均衡策略。Horizon 会不断地检查队列的繁忙程度，并将一些 worker 从不繁忙的队列中抽出来分配给繁忙的队列去处理任务，直到这些队列都恢复平静。

在任何时候，每个进程池至少具有在配置文件中添加的最少进程数，这数字至少要为 1，默认情况下的设置也是为 1。

# 主监督者

主监督者的工作是启动监督者，并确保它们继续做好自己的工作。当你运行 `horizon` 命令时，你的配置文件会被扫描，另外 `AddSupervisor` 任务会被分配到每个需要添加监督者的特殊队列中去。主监督者会不断检查这个队列，并让其中的任务在主进程的每个循环上执行。

在开始循环之前，注册了很多 pcntl 信号监听器，Horizon 使用这些信号来确认何时重启、暂停、终止或者继续暂停状态。

```Php
pcntl_signal(SIGTERM, function () {
    $this->terminate();
});

pcntl_signal(SIGUSR1, function () {
    $this->restart();
});

pcntl_signal(SIGUSR2, function () {
    $this->pause();
});

pcntl_signal(SIGCONT, function () {
    $this->continue();
});
```

在循环内部，主处理器处理其特殊队列中的所有挂起的任务，监视子监督者的进程，并在每次迭代后触发 `MasterSupervisorLooped` 事件。

此时 `AddSupervisor` 命令会被推送到主队列运行，为每个监督者创建了一个运行 `php artisan horizon:supervisor` 命令的 Symfony 进程，这个命令通过调用名为 `SupervisorProcess` 的类，来确保 Symfony 进程始终在运行。最后，`SupervisorProcess` 的实例存储在 `MasterSupervisor` 实例的 `supervisors` 数组的属性中。

# 启动监督者

当主监督者循环时，Horizon 将在所有 `SupervisorProcess` 实例上调用 `monitor()` 方法，这个方法确保了 `horizon:supervisor` 程序进程正在运行，它还负责重新启动和终止，以及在配置文件更改时重新配置它。

The `horizon:Supervisor` command creates a `Supervisor` instance and starts creating the processes pools used to run & monitor this supervisor workers, the configuration of process pools depends on your provided balancing strategy.

 `horizon:Supervisor` 命令创建一个 `Supervisor` 实例，并开始创建进程池用于运行和监视这个监督者下面的 workers，进程池的配置取决于你提供的均衡策略。

### 监督者初始行为

一开始 Horizon 会检查你是否使用任何类型的均衡，如果有，它会配置监督者所处理的每个队列创建一个进程池。创建进程池后，Horizon 将通过将总进程数除以池数来为每个池创建多个 `WorkerProcess`。

# 运行 supervisor 循环

一旦监督者准备好运行，Horizon 就会注册一些 pcntl 信号监听器并启动一个循环，它会执行以下操作：

- 处理任何与 Supervisor 相关的命令。
- 开始监控每个进程池。
- 如果有正在运行的自动均衡策略，就处理均衡。

> 每个监督者都有一个特殊队列，Horizon 会推送任务来执行它。例如当 Horizon 希望监督者终止时，便会推送一个 `Terminate` 命令并运行，以确保监督者安全终止其进程。

每个进程池会遍历 worker 的进程并确保它们正在运行，使用  `php artisan queue:work`  命令，就能让 worker 作为守护进程在后台运行。