# Workers

Now that we learnt how Laravel pushes jobs into different queues, let's perform a dive into how workers run your jobs. First I'd like to define workers as a simple PHP process that runs in the background with the purpose of extracting jobs from a storage space and run them with respect to several configuration options.

```
php artisan queue:work
```

Running this command will instruct Laravel to create an instance of your application and start executing jobs, this instance will stay alive indefinitely which means the action of starting your Laravel application happens only once when the command was run & the same instance will be used to execute your jobs, that means the following:

- You save server resources by avoiding booting up the whole app on every job.
- You have to manually restart the worker to reflect any code change you made in your application.

You can also run:

```
php artisan queue:work --once
```

This will start an instance of the application, process a single job, and then kill the script.

```
php artisan queue:listen
```

The `queue:listen` command simply runs the `queue:work --once` command inside an infinite loop, this will cause the following:

- An instance of the app is booted up on every loop.
- The assigned worker will pick a single job and execute it.
- The worker process will be killed.

Using `queue:listen` ensures that a new instance of the app is created for every job, that means you don't have to manually restart the worker in case you made changes to your code, but also means more server resources will be consumed.

# The queue:work command

Let's take a look at the `handle()` method of the `Queue\Console\WorkCommand` class, it's the method that'll be executed when you run `php artisan queue:work`:

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

First, we check if the application is in maintenance mode & the `--once` option is used, in that case we want the script to die gracefully so we don't execute any jobs, for that reason we'll just ask the worker to sleep for a given period before killing the script entirely.

Here's how the `sleep()` method of `Queue\Worker` looks like:

```
public function sleep($seconds)
{
    sleep($seconds);
}
```

#### Why don't we just return null in the handle() method to kill the script?

As we said earlier the `queue:listen` command runs the `WorkCommand` inside a loop:

```
while (true) {
     // This process simply calls 'php artisan queue:work --once'
    $this->runProcess($process, $options->memory);
}
```

If the app is in maintenance mode and the `WorkCommand` terminated immediately this will cause the loop to end and a next one starts in a very short time, it's better that we cause some delay in that case instead of consuming the server resources by creating tons of application instances that we won't really use.

## Listening for events

Inside the `handle()` method we call the `listenForEvents()` method:

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

In this method we listen to several events our workers will fire down the road, this will allow us to print some information to the user every time a job is being processed, passed, or failed.

## Logging failed jobs

In case a job fails, the `logFailedJob()` method is called:

```
$this->laravel['queue.failer']->log(
    $event->connectionName, $event->job->getQueue(),
    $event->job->getRawBody(), $event->exception
);
```

The `queue.failer` container alias is registered in `Queue\QueueServiceProvider::registerFailedJobServices()`:

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
 * Create a new database failed job provider.
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

In case the `queue.failed` configuration value is set, the database queue failer will be used and it simply stores information about the failed job in a database table:

```
$this->getTable()->insertGetId(compact(
    'connection', 'queue', 'payload', 'exception', 'failed_at'
));
```

## Running the worker

To run the worker we need to collect two pieces of information:

- The connection this worker will be pulling jobs from
- The queue the worker will use to find jobs

You can provide a `--connection=default` option to the `queue:work` command, but in case you didn't the default collection defined in the `queue.default` configuration value will be used.

Same for the queue, you can provide a `--queue=emails` option or the `queue` option set in your selected connection configuration will be used.

Once all this is done, the `WorkCommand::handle()` method runs `runWorker()`:

```
protected function runWorker($connection, $queue)
{
    $this->worker->setCache($this->laravel['cache']->driver());

    return $this->worker->{$this->option('once') ? 'runNextJob' : 'daemon'}(
        $connection, $queue, $this->gatherWorkerOptions()
    );
}
```

The worker class property is set while the command is being constructed:

```
public function __construct(Worker $worker)
{
    parent::__construct();

    $this->worker = $worker;
}
```

The container resolves an instance of `Queue\Worker`, inside `runWorker()` we set the cache driver the worker will use, we also decide what method we'll call to based on the `--once` command option.

In case the `--once` option is used we'll just call `runNextJob` to run the next available job and then the script dies. Otherwise we'll call the `daemon` method which will keep the process alive processing jobs all the time.

While starting the worker we gather the command options given by the user using the `gatherWorkerOptions()` method, we later provide these options the worker `runNextJob` or `daemon` methods.

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

## The Daemon

Let's take a look at the `Worker::daemon()` method, first line in this method calls the `listenForSignals()` method:

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

This method uses PHP7.1's signal handlers, the `supportsAsyncSignals()` method checks if we're on PHP7.1 and that the `pcntl` extension is loaded.

`pcntl_async_signals()` is later called to enable signal handling, then we register handlers for multiple signals:

- `SIGTERM` is raised when the script is instructed to shutdown.
- `SIGUSR2` is a user-defined signal Laravel uses to indicate that the script should pause.
- `SIGCONT` is raised when a paused script should proceed.

These signals are sent from a Process Monitor such as [Supervisor](http://supervisord.org/) to communicate with our script.

Second line in the `Worker::daemon()` method fetches the timestamp of last queue restart, this value is stored in the cache when we call the `queue:restart`, later on we'll check if the timestamp of last restart doesn't match which indicates that the worker should restart, more on that later.

Finally the method starts a loop where we'll do the rest of the work of fetching jobs, running them, and do several actions on the worker process.

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

### Determining if the worker should process jobs

Calling `daemonShouldRun()` we check for the following cases:

- Application is not in maintenance mode
- Worker is not paused
- No event listeners preventing the loop from continuing

If app in maintenance mode you can still process jobs if your worker run with the `--force` option:

```
php artisan queue:work --force
```

One of the conditions determining if the worker should continue is the following:

```
$this->events->until(new Events\Looping($connectionName, $queue)) === false)
```

This line fires a `Queue\Event\Looping` event and checks if any of the listeners return false in its `handle()` method, using this fact you can occasionally force your workers to stop processing jobs temporarily.

In case the worker should pause, the `pauseWorker()` method is called:

```
protected function pauseWorker(WorkerOptions $options, $lastRestart)
{
    $this->sleep($options->sleep > 0 ? $options->sleep : 1);

    $this->stopIfNecessary($options, $lastRestart);
}
```

This method calls the `sleep` method and passes the `--sleep` option given to the console command:

```
public function sleep($seconds)
{
    sleep($seconds);
}
```

After the script sleeps for a while, we check if the worker should quit and kill the script in that case, we'll look into the `stopIfNecessary` method later, in case the script shouldn't be killed we'll just call `continue;` to start a new loop:

```
if (! $this->daemonShouldRun($options, $connectionName, $queue)) {
    $this->pauseWorker($options, $lastRestart);

    continue;
}
```

### Retrieving a job to run

```
$job = $this->getNextJob(
    $this->manager->connection($connectionName), $queue
);
```

The `getNextJob()` method accepts an instance of the desired queue connection and the queue we should fetch jobs from:

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

We simply loop on the given queue(s), use the selected queue connection to get a job from the storage space (database, redis, sqs, ...) and return that job.

To retrieve a job from storage we query for the oldest job that meets the following conditions:

- Pushed to the `queue` we're trying to find jobs within
- Not reserved by another worker
- Available to be run at the given time, some jobs are delayed to run in the future
- We also get jobs that are reserved for a long time they became frozen so we retry them

Once we find a job that meets this criteria we mark this job as reserved so that other workers don't pick it up, we also increment the number of attempts of the job for monitoring.

### Monitoring jobs timeout

After the next job is retrieved, we call the `registerTimeoutHandler()` method:

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

Again if the `pcntl` extension is loaded, we'll register a signal handler that kills the worker process if the job timed out, we use `pcntl_alarm()` to send a `SIGALRM` signal after the configured timeout is passed.

If the job took longer than the timeout value the handler will kill the script, if not the job will pass and the next loop will set a new alarm overriding the first one since a single alarm can be present in the process.

Jobs timeout only work on PHP7.1 or above, it also doesn't work on Windows ¯\_(ツ)_/¯

### Processing a job

The `runJob()` method calls `process()`:

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

Here `raiseBeforeJobEvent()` fires the `Queue\Events\JobProcessing` event, and `raiseAfterJobEvent()` fires the `Queue\Events\JobProcessed` event.

`markJobAsFailedIfAlreadyExceedsMaxAttempts()` checks if the process already reached the maximum attempts and mark the job as failed in that case:

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

Otherwise we call the `fire()` method on the job object to run the job.

#### Where did we get this job object?

The `getNextJob()` method returns an instance of `Contracts\Queue\Job`, depends on the queue driver we use the respective Job instance will be used, for example `Queue\Jobs\DatabaseJob` in case the Database queue driver is selected.

### End of loop

At the end of the loop we call `stopIfNecessary()` to check if we should kill the process before the next loop starts:

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

The `shouldQuit` property is set in two situations, first as a signal handler for the `SIGTERM` signal set inside `listenForSignals()`, second inside `stopWorkerIfLostConnection()`:

```
protected function stopWorkerIfLostConnection($e)
{
    if ($this->causedByLostConnection($e)) {
        $this->shouldQuit = true;
    }
}
```

This method is called in several try...catch statements while retrieving and processing jobs to ensure that the worker should die so that our Process Control may start a new one with a fresh database connection.

The `causedByLostConnection()` method can be found in the `Database\DetectsLostConnections` trait.

`memoryExceeded()` checks if memory usage exceeded the current set memory limit, you can set the limit using the `--memory` option on the `queue:work` command.

Finally `queueShouldRestart()` compares the current timestamp of a restart signal and if it doesn't match the time we stored while starting the worker process this means that a new restart signal was sent during the loop, in that case we kill the process so that it can be restarted by the Process Control later.