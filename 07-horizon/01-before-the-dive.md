#Before The Dive

Laravel Horizon is a queue manager that gives you full control over your queues, it provides means to configure how your jobs are processed, generate analytics, and perform different queue-related tasks from within a nice dashboard.

In this dive we're going to learn how Horizon boots up and handles processing jobs using different workers as well as how it collects useful metrics for you to have the full picture of how your application dispatches and runs jobs.

It all starts by running `php artisan horizon`, this command scans your `horizon.php`configuration file & starts a number of queue workers based on these configurations:

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
        balance=> "simple", // could be simple, auto, or null
        force=> false,
    ],
    'supervisor-2' => [
        // ...
    ]
],
```

Here you may define any number of supervisors for each environment, a supervisor is a process living in your system with the purpose of monitoring a number of Queue Workers and make sure they're working as configured, for every supervisor you can set the following:

1. By default a supervisor monitors your `default` queue but you can add any number of queues.
2. Minimum & Maximum number of Workers allowed to run.
3. Amount of time to delay failed jobs.
4. Memory limit of every worker.
5. Timeout for every worker.
6. Number of seconds to sleep when no job is available.
7. Number of times to attempt a job before logging it failed.
8. The balancing strategy.
9. Wether your workers should run in maintenance mode or not.

#### What is a balancing strategy?

You have three options:

1. No balancing.
2. Simple Balancing.
3. Auto Balancing.

In case you don't want to balance, a supervisor will have a single pool of workers processes running jobs from all the supervised queues at the same time.

However in case you chose `simple` as your balancing strategy, Horizon will create a process pool for every queue, so for example if your supervisor should be working on the `emails` & `notifications` queues and you set the number of processes to be 4, Horizon will create two process pools each one monitoring 2 worker processes.

If you want Horizon to distribute the load based on how busy a queue is, choose the `auto` balancing strategy and Horizon will continuously check how busy your queues are and move some workers assigned to a non-busy queue to process jobs from another busy queue until things are back to normal.

At all times every process pool will have at least the minimum number of processes you added in your configuration file, this number can be at least 1 and it's set to 1 by default.

# The Master Supervisor

The Master's job is to start the supervisors and make sure they keep doing their job, when you run the `horizon` command your configuration file is scanned and an `AddSupervisor` job is dispatched to a special queue for every supervisor that needs to be added, this queue is constantly checked by the Master and the jobs inside it are executed on every loop of the master process.

Before starting the loop a number of pcntl signal listeners are registered, Horizon uses these signals to know when to restart, pause, terminate, or continue a paused state.

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

Inside the loop, the master processes any pending jobs in its special queue, monitors child supervisor processes, and fires the `MasterSupervisorLooped` event after each iteration.

The `AddSupervisor` command that gets pushed to the master queue is ran at this point, it simply creates a new Symfony Process that runs `php artisan horizon:supervisor` for each supervisor, this process is passed to a class called `SupervisorProcess` whose job is to ensure that Symfony process is always running. Finally that instance of `SupervisorProcess` is stored into a `supervisors` array property on the `MasterSupervisor` instance.

# Starting Supervisors

While the master supervisor is looping, Horizon will call the `monitor()` method on all these `SupervisorProcess` instances, this method makes sure the `horizon:supervisor` process is running, it also takes care of restarting & terminating it as well as re-provisioning it in case the configuration file was changed.

The `horizon:Supervisor` command creates a `Supervisor` instance and starts creating the processes pools used to run & monitor this supervisor workers, the configuration of process pools depends on your provided balancing strategy.

### Initial scaling of a supervisor

Initially Horizon will check if you use any kind of balancing, if so it'll create a process pool for each of the queues the supervisor is configured to work on, once the process pools are created Horizon will create multiple `WorkerProcess` for each pool by dividing the number of total processes over the number of pools.

# Running the supervisor loop

Once the supervisor is ready to run, Horizon registers some pcntl signal listeners and starts a loop where it does the following:

- Processes any Supervisor related commands.
- Starts Monitoring each process pool.
- Handles balancing if we're running the auto-balancing strategy.

Each supervisor has a special queue too where Horizon pushes jobs for it to execute, an example is when Horizon wants a Supervisor to terminate it pushed a `Terminate` command and the supervisor runs it to ensure safe termination of the process.

Each process pool will loop over its worker processes and make sure they're running, a worker process is simply a `php artisan queue:work` command running as a daemon in the background.