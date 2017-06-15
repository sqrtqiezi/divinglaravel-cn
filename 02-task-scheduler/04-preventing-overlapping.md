#  Preventing Overlapping

Sometimes a scheduled job takes more time to run than what we initially expected, and this causes another instance of the job to start while the first one is not done yet, for example imagine that we run a job that generates a report every minute, after sometime when the data gets huge the report generation might take more than 1 minute so another instance of that job starts while the first is still ongoing.

In most scenarios this is fine, but sometimes this should be prevented in order to guarantee correct data or prevent a high server resources consumption, so let's see how you can prevent such scenario in laravel:

```php
$schedule->command('mail:send')->withoutOverlapping();
```

Laravel will check for the `Console\Scheduling\Event::withoutOverlapping` class property and if it's set to true it'll try to create a mutex for the job, and will only run the job if creating a mutex was possible.

### But what's a mutex?

Here's the most interesting explanation I could find online:

> When I am having a big heated discussion at work, I use a rubber chicken which I keep in my desk for just such occasions. The person holding the chicken is the only person who is allowed to talk. If you don't hold the chicken you cannot speak. You can only indicate that you want the chicken and wait until you get it before you speak. Once you have finished speaking, you can hand the chicken back to the moderator who will hand it to the next person to speak. This ensures that people do not speak over each other, and also have their own space to talk. Replace Chicken with Mutex and person with thread and you basically have the concept of a mutex.
>
> -- https://stackoverflow.com/questions/34524/what-is-a-mutex/34558#34558

So Laravel creates a mutex when the job starts the very first time, and then every time the job runs it checks if the mutex exists and only runs the job if it doesn't.

Here's what happens inside the `withoutOverlapping` method:

```php
public function withoutOverlapping()
{
    $this->withoutOverlapping = true;

    return $this->then(function () {
        $this->mutex->forget($this);
    })->skip(function () {
        return $this->mutex->exists($this);
    });
}
```

So Laravel creates a filter-callback method that instructs the Schedule Manager to ignore the task if a mutex still exists, it also creates an after-callback that clears the mutex after an instance of the task is done.

Also before running the job, Laravel does the following check inside the `Console\Scheduling\Event::run()` method:

```php
if ($this->withoutOverlapping && ! $this->mutex->create($this)) {
    return;
}
```

### Where does the mutex property come from?

While the instance of `Console\Scheduling\Schedule` is being instantiated, laravel checks if an implementation to the `Console\Scheduling\Mutex` interface was bound to the container, if yes it uses that instance but if not it uses an instance of `Console\Scheduling\CacheMutex`:

```php
$this->mutex = $container->bound(Mutex::class)
                        ? $container->make(Mutex::class)
                        : $container->make(CacheMutex::class);
```

Now while the Schedule Manager is registering your events it'll pass an instance of the mutex:

```php
$this->events[] = new Event($this->mutex, $command);
```

> By default Laravel uses a cache-based mutex, but you can override that and implement your own mutex approach & bind it to the container.

## The cache-based mutex

The `CacheMutex` class contains 3 simple methods, it uses the event mutex name as a cache key:

```php
public function create(Event $event)
{
    return $this->cache->add($event->mutexName(), true, 1440);
}

public function exists(Event $event)
{
    return $this->cache->has($event->mutexName());
}

public function forget(Event $event)
{
    $this->cache->forget($event->mutexName());
}
```

## Mutex removal after task finishes

As we've seen before, the manager registers an after-callback that removes the mutex after the task is done, for a task that runs a command on the OS that might be enough to ensure that the mutex is cleared, but for a callback task the script might die while executing the callback, so to prevent that an extra fallback was added in `Console\Scheduling\CallbackEvent::run()`:

```php
register_shutdown_function(function () {
    $this->removeMutex();
});
```

This removes the mutex in case the script shuts down un-expectedly.