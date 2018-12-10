# Workers

现在我们已经了解了 Laravel 如何将任务推进不同的队列，接下来我们继续深入了解 worker 是如何运作的。首先，我想将 worker 定义为一个简单的PHP进程，它在后台运行，目的是从存储空间中提取任务，并根据几个配置选项运行它们。

```
php artisan queue:work
```

运行这个命令后，Laravel 会在你的应用程序上创建实例并开始执行任务，这个实例会将无限期保持活动状态，这意味着启动这个命令的操作仅在 Laravel 应用程序中发生一次，并且将使用相同的实例执行你的任务，这意味着以下几点：

- 你要避免在每个任务上启动整个应用程序来节省服务器资源。
- 你必须手动重新启动 worker 让使你在应用程序中所做的任何代码的更改生效。

你也可以运行：

```
php artisan queue:work --once
```

这会启动应用程序的实例，处理单个任务，然后终止该脚本。

```
php artisan queue:listen
```

`queue:listen` 命令只是在无限循环中运行 `queue:work --once` 命令，这将导致以下结果：

- 每个循环都会启动应用程序的一个实例。
- 被分配的 worker 会选择一个任务并执行它。
- 工作进程将被杀死。

使用 `queue:listen` 确保为每个任务创建应用程序的新实例，这意味着你不必因为更改代码而手动重新启动 worker，同时这也意味着会消耗更多的服务器资源。

# queue:work 命令

让我们看一下  `Queue\Console\WorkCommand` 类的 `handle()` 方法，它是在运行 `php artisan queue:work` 时执行的方法：

```
public function handle()
{
    if ($this->downForMaintenance() && $this->option('once')) {
        return $this->worker->sleep($this->option('sleep'));
    }

    $this->listenForEvents();

    $connection = $this->argument('connection')
                    ?: $this->laravel['config']['queue.default'];

    $queue = $this->getQueue($connection);

    $this->runWorker(
        $connection, $queue
    );
}
```

首先，我们检查应用程序是否处于维护模式并使用 `--once` 选项。在这种情况下，我们希望不执行任何任务，脚本能优雅地死亡，因此我们只需要求 worker 在完全杀死脚本之前睡眠一段时间。

以下是 `Queue\Worker` 的 `sleep()` 方法如下所示：

```
public function sleep($seconds)
{
    sleep($seconds);
}
```

#### 为什么不在 handle() 方法中返回 null 来杀死脚本？ 

正如我们之前所说的 `queue:listen` 命令是在循环中运行 `WorkCommand`：

```
while (true) {
     // 这个过程简单地调用了 'php artisan queue:work --once'
    $this->runProcess($process, $options->memory);
}
```

如果应用程序处于维护模式时 `WorkCommand` 立即终止，会导致循环结束并且下一个循环会在很短的时间内启动。最好的做法是我们在这种情况下造成一些延迟，而不是通过创建大量我们不会真正使用的应用程序实例来消耗服务器资源。

## 监听事件

在 `handle()` 中我们调用 `listenForEvents()` 方法：

```
protected function listenForEvents()
{
    $this->laravel['events']->listen(JobProcessing::class, function ($event) {
        $this->writeOutput($event->job, 'starting');
    });

    $this->laravel['events']->listen(JobProcessed::class, function ($event) {
        $this->writeOutput($event->job, 'success');
    });

    $this->laravel['events']->listen(JobFailed::class, function ($event) {
        $this->writeOutput($event->job, 'failed');

        $this->logFailedJob($event);
    });
}
```

在这个方法中我们监听几个会导致 worker 停止运行的事件，有了这个我们就能在每次处理、传递或失败作业时向用户打印一些信息。

## 记录失败任务

如果任务失败，就会调用 `logFailedJob()` 方法：

```
$this->laravel['queue.failer']->log(
    $event->connectionName, $event->job->getQueue(),
    $event->job->getRawBody(), $event->exception
);
```

`queue.failer` 容器别名注册在 `Queue\QueueServiceProvider::registerFailedJobServices()`：

```
protected function registerFailedJobServices()
{
    $this->app->singleton('queue.failer', function () {
        $config = $this->app['config']['queue.failed'];

        return isset($config['table'])
                    ? $this->databaseFailedJobProvider($config)
                    : new NullFailedJobProvider;
    });
}

/**
 * 创建新的 databaseFailedJobProvider
 *
 * @param  array  $config
 * @return \Illuminate\Queue\Failed\DatabaseFailedJobProvider
 */
protected function databaseFailedJobProvider($config)
{
    return new DatabaseFailedJobProvider(
        $this->app['db'], $config['database'], $config['table']
    );
}
```

如果设置了 `queue.failed` 配置值，将使用数据库队列 failer，它只将失败任务的信息存储在数据库表中：

```
$this->getTable()->insertGetId(compact(
    'connection', 'queue', 'payload', 'exception', 'failed_at'
));
```

## 运行 worker

要运行 worker，我们需要收集两条信息：

- worker 拉取任务的连接
- worker 用于查找任务的队列

你可以为 `queue:work` 命令提供 `--connection=default` 选项。如果没有，则将使用 `queue.default` 配置值中定义的默认集合。

至于队列，你可以提供 `--queue=emails` 选项，否则会使用在所选连接配置中设置的 `queue` 选项。

完成所有这些之后，`WorkCommand::handle()` 方法会运行 `runWorker()`：

```
protected function runWorker($connection, $queue)
{
    $this->worker->setCache($this->laravel['cache']->driver());

    return $this->worker->{$this->option('once') ? 'runNextJob' : 'daemon'}(
        $connection, $queue, $this->gatherWorkerOptions()
    );
}
```

在构造函数中设置 worker 类属性：

```
public function __construct(Worker $worker)
{
    parent::__construct();

    $this->worker = $worker;
}
```

容器解析了 `Queue\Worker` 的实例，在 `runWorker()` 中我们设置了 worker 使用的缓存驱动程序，我们还根据 `--once` 命令选项决定我们将调用哪种方法。

如果使用 `--once` 选项，我们只需调用 `runNextJob` 来运行下一个可用任务，然后脚本就会死掉。否则，我们将调用 `daemon` 方法，这个方法将始终保持进程处理状态。

在启动 worker 时，我们使用 `gatherWorkerOptions()` 方法收集用户给出的命令选项，之后我们会为 worker 的 `runNextJob` or `daemon` 方法提供这些选项。

```
protected function gatherWorkerOptions()
{
    return new WorkerOptions(
        $this->option('delay'), $this->option('memory'),
        $this->option('timeout'), $this->option('sleep'),
        $this->option('tries'), $this->option('force')
    );
}
```

## 守护进程

让我们看一下 `Worker::daemon()` 方法的第一行调用的 `listenForSignals()` 方法：

```
protected function listenForSignals()
{
    if ($this->supportsAsyncSignals()) {
        pcntl_async_signals(true);

        pcntl_signal(SIGTERM, function () {
            $this->shouldQuit = true;
        });

        pcntl_signal(SIGUSR2, function () {
            $this->paused = true;
        });

        pcntl_signal(SIGCONT, function () {
            $this->paused = false;
        });
    }
}
```

这个方法使用 PHP7.1 的信号处理程序，`supportsAsyncSignals()` 方法检查我们是否使用 PHP7.1 并且加载了 `pcntl` 扩展。

接下来调用的 `pcntl_async_signals()` 来启用信号处理，然后我们为多个信号注册处理程序：

- 指示脚本关闭时会引发 `SIGTERM`。
- `SIGUSR2` 是 Laravel 用来指示脚本应该暂停的用户定义信号。
- 当暂停的脚本继续时，会引发 `SIGCONT`。

这些信号从 [Supervisor](http://supervisord.org/) 等进程监视器发送与我们的脚本进行通信。

`Worker::daemon()` 方法中的第二行获取上次队列重启的时间戳，当我们调用 `queue:restart` 时，该值存储在缓存中，然后就检查上次重启的时间戳是否不匹配，这表明 worker 应该重新启动。稍后再详细说明一下~

最后，该方法启动一个循环，我们将完成其他工作，即获取任务、运行它们，并对 worker 进程执行多项操作。

```
while (true) {
    if (! $this->daemonShouldRun($options, $connectionName, $queue)) {
        $this->pauseWorker($options, $lastRestart);

        continue;
    }

    $job = $this->getNextJob(
        $this->manager->connection($connectionName), $queue
    );

    $this->registerTimeoutHandler($job, $options);

    if ($job) {
        $this->runJob($job, $connectionName, $options);
    } else {
        $this->sleep($options->sleep);
    }

    $this->stopIfNecessary($options, $lastRestart);
}
```

### 决定 worker 是否应该处理任务

要调用 `daemonShouldRun()` 我们得先检查以下情况：

- 应用程序未处于维护模式
- Worker 没有暂停
- 没有事件监听器阻止循环继续

如果 app 处于维护模式，如果 worker 使用 `--force` 选项运行，你仍然可以处理任务：

```
php artisan queue:work --force
```

确定 worker 是否应该继续的条件之一是：

```
$this->events->until(new Events\Looping($connectionName, $queue)) === false)
```

这行代码触发 `Queue\Event\Looping` 事件，用来检查是否有任何监听器在其 `handle()` 方法中返回 false。利用这个方法可以强制 worker 暂时停止处理作业。

如果 worker 应该暂停，则调用 `pauseWorker()` 方法： 

```
protected function pauseWorker(WorkerOptions $options, $lastRestart)
{
    $this->sleep($options->sleep > 0 ? $options->sleep : 1);

    $this->stopIfNecessary($options, $lastRestart);
}
```

这个方法调用 `sleep` 并传递给控制台命令的 `--sleep`：

```
public function sleep($seconds)
{
    sleep($seconds);
}
```

在脚本休眠一段时间之后，我们检查 worker 是否应该退出并在这种情况下终止脚本。 `stopIfNecessary` 方法等下再看。假设脚本不应该被杀死，我们只需要调用 `continue;` 开始一个新的循环：

```
if (! $this->daemonShouldRun($options, $connectionName, $queue)) {
    $this->pauseWorker($options, $lastRestart);

    continue;
}
```

### 检索要运行的作业

```
$job = $this->getNextJob(
    $this->manager->connection($connectionName), $queue
);
```

`getNextJob()` 方法接受所需队列连接的实例以及我们应从中获取任务的队列：

```
protected function getNextJob($connection, $queue)
{
    try {
        foreach (explode(',', $queue) as $queue) {
            if (! is_null($job = $connection->pop($queue))) {
                return $job;
            }
        }
    } catch (Exception $e) {
        $this->exceptions->report($e);

        $this->stopWorkerIfLostConnection($e);
    }
}
```

我们只是循环使用给定的队列，使用选定的队列连接从存储空间（database、redis、sqs……）获取任务并返回任务。

要从存储中检索作业，我们会查询满足以下条件的最早的任务：

- 推到 `queue` 我们正在努力找的任务
- 不是由另一个 woker 保留的
- 可以在给定时间运行，某些作业将在未来延迟运行
- 我们也获得了长期保留的工作，他们被冻结，所以我们重试它们

一旦我们找到符合此条件的任务，我们会将此任务标记为已保留，这样不会被其他 worker 提取，我们也会增加该任务的尝试次数以进行监控。

### 监控任务超时

检索下一个任务后，我们调用 `registerTimeoutHandler()` 方法：

```
protected function registerTimeoutHandler($job, WorkerOptions $options)
{
    if ($this->supportsAsyncSignals()) {
        pcntl_signal(SIGALRM, function () {
            $this->kill(1);
        });the

        $timeout = $this->timeoutForJob($job, $options);

        pcntl_alarm($timeout > 0 ? $timeout + $options->sleep : 0);
    }
}
```

如果加载 `pcntl` 扩展，我们将注册一个信号处理程序。如果任务超时，则杀死工作进程。我们使用 `pcntl_alarm()` 发送 `SIGALRM` 信号来传递配置超时。

如果任务花费的时间长于超时值，则处理程序会终止该脚本。如果不是，任务将通过并且在下一个循环设置覆盖第一个的新警报，因为在该过程中可以存在单个警报。

> 任务超时仅适用于 PHP7.1 或更高版本，它也不适用于 Windows ¯_(ツ)_/¯

### 处理任务

`runJob()` 方法调用 `process()`:

```
public function process($connectionName, $job, WorkerOptions $options)
{
    try {
        $this->raiseBeforeJobEvent($connectionName, $job);

        $this->markJobAsFailedIfAlreadyExceedsMaxAttempts(
            $connectionName, $job, (int) $options->maxTries
        );

        $job->fire();

        $this->raiseAfterJobEvent($connectionName, $job);
    } catch (Exception $e) {
        $this->handleJobException($connectionName, $job, $options, $e);
    }
}
```

这里 `raiseBeforeJobEvent()` 触发 `Queue\Events\JobProcessing` 事件，`raiseAfterJobEvent()` 触发 `Queue\Events\JobProcessed` 事件。

`markJobAsFailedIfAlreadyExceedsMaxAttempts()` 检查进程是否已达到最大重试次数，而且在这种情况下会讲任务标记为失败：

```
protected function markJobAsFailedIfAlreadyExceedsMaxAttempts($connectionName, $job, $maxTries)
{
    $maxTries = ! is_null($job->maxTries()) ? $job->maxTries() : $maxTries;

    if ($maxTries === 0 || $job->attempts() <= $maxTries) {
        return;
    }

    $this->failJob($connectionName, $job, $e = new MaxAttemptsExceededException(
        'A queued job has been attempted too many times. The job may have previously timed out.'
    ));

    throw $e;
}
```

否则，我们在任务对象上调用 `fire()` 方法来运行任务。

#### 那我们从哪里得到这个任务对象？

`getNextJob()` 方法返回 `Contracts\Queue\Job` 的实例，取决于我们使用的队列驱动程序使用相应的任务实例。例如，如果选择了数据库队列驱动程序，则为 `Queue\Jobs\DatabaseJob`。

### 循环结束

在循环结束时，我们调用 `stopIfNecessary()` 来检查是否应该在下一个循环开始之前终止进程：

```
protected function stopIfNecessary(WorkerOptions $options, $lastRestart)
{
    if ($this->shouldQuit) {
        $this->kill();
    }

    if ($this->memoryExceeded($options->memory)) {
        $this->stop(12);
    } elseif ($this->queueShouldRestart($lastRestart)) {
        $this->stop();
    }
}
```

`shouldQuit` 属性设置在两种情况下使用，一个作为 `listenForSignals()` 内 `SIGTERM` 信号集的信号处理程序，另一个在 `stopWorkerIfLostConnection()` 中：

```
protected function stopWorkerIfLostConnection($e)
{
    if ($this->causedByLostConnection($e)) {
        $this->shouldQuit = true;
    }
}
```

在检索和处理作业时，在几个 try … catch 语句中调用此方法，以确保 worker 程序应该被杀死，以便我们的过程控制可以使用新的数据库连接启动新的过程控制。

`causedByLostConnection()` 方法能在 `Database\DetectsLostConnections` trait 中被找到。

`memoryExceeded()` 检查内存使用量是否超过当前设置的内存限制，可以使用 `queue:work` 命令中的 `--memory` 选项设置限制。

最后 `queueShouldRestart()` 比较重启信号的当前时间戳，如果它与我们在启动工作进程时存储的时间不匹配，这意味着在循环期间发送了一个新的重启信号，在这种情况下，我们终止进程，以便稍后由过程控制重新启动它。