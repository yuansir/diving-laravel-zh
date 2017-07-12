Sometimes a scheduled job takes more time to run than what we initially expected, and this causes another instance of the job to start while the first one is not done yet, for example imagine that we run a job that generates a report every minute, after sometime when the data gets huge the report generation might take more than 1 minute so another instance of that job starts while the first is still ongoing.

有时一个预定的工作需要比我们最初预期的更多的时间运行，这样会导致另外一个工作的实例开始，而第一个还没有完成，例如，我们运行一个每分钟生成报告的工作有时候当数据变大时，报表生成可能需要1分钟以上，这样就可以在第一个还在进行时启动该作业的另一个实例。

In most scenarios this is fine, but sometimes this should be prevented in order to guarantee correct data or prevent a high server resources consumption, so let's see how you can prevent such scenario in laravel:

在大多数情况下，这是很好的，但有时候应该防止这种情况，以保证正确的数据或防止高的服务器资源消耗，所以让我们看看如何防止这种情况在laravel中发生：

```php
$schedule->command('mail:send')->withoutOverlapping();
```

Laravel will check for the `Console\Scheduling\Event::withoutOverlapping` class property and if it's set to true it'll try to create a mutex for the job, and will only run the job if creating a mutex was possible.

Laravel将检查 `Console\Scheduling\Event::withoutOverlapping` 类属性，如果设置为true，它将尝试为作业创建互斥，并且只有在创建互斥的情况下才能运行该作业。

#### But what's a mutex?

#### 但是上面是互斥?

Here's the most interesting explanation I could find online:

这是我可以在网上找到最有趣的解释：

> When I am having a big heated discussion at work, I use a rubber chicken which I keep in my desk for just such occasions. The person holding the chicken is the only person who is allowed to talk. If you don't hold the chicken you cannot speak. You can only indicate that you want the chicken and wait until you get it before you speak. Once you have finished speaking, you can hand the chicken back to the moderator who will hand it to the next person to speak. This ensures that people do not speak over each other, and also have their own space to talk. Replace Chicken with Mutex and person with thread and you basically have the concept of a mutex.
>
> -- [https://stackoverflow.com/questions/34524/what-is-a-mutex/34558#34558](https://stackoverflow.com/questions/34524/what-is-a-mutex/34558#34558)

> 当我在工作中进行热烈的讨论时，我使用一只橡胶鸡，我在这样的场合放在桌子上。 持有鸡的人是唯一被允许谈话的人。 如果你不握鸡，你不会说话。 你只能指示你想要鸡，等到你说话之前才能得到它。 一旦你完成演讲，你可以将鸡回到主持人，他将把它交给下一个人说话。 这样可以确保人们互不说话，也有自己的空间。 用线替换鸡与互斥和人，你基本上有一个互斥的概念。
>
> -- [https://stackoverflow.com/questions/34524/what-is-a-mutex/34558#34558](https://stackoverflow.com/questions/34524/what-is-a-mutex/34558#34558)

So Laravel creates a mutex when the job starts the very first time, and then every time the job runs it checks if the mutex exists and only runs the job if it doesn't.

所以当作业第一次启动时，Laravel创建一个互斥，然后每次作业运行时，它检查互斥是否存在，只有在没有工作的情况下运行。

Here's what happens inside the `withoutOverlapping` method:

这里是 `withoutOverlapping` 方法中做的事

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

因此，Laravel创建一个filter-callback方法，指示Schedule Manager忽略任务，如果互斥仍然存在，它还会创建一个在完成任务实例后清除互斥的回调。

Also before running the job, Laravel does the following check inside the `Console\Scheduling\Event::run()` method:

在运行该作业之前，Laravel会在`Console\Scheduling\Event::run()`方法中进行以下检查：

```php
if ($this->withoutOverlapping && ! $this->mutex->create($this)) {
    return;
}
```

#### Where does the mutex property come from?

#### 互斥体属性来自哪里？

While the instance of `Console\Scheduling\Schedule` is being instantiated, laravel checks if an implementation to the `Console\Scheduling\Mutex` interface was bound to the container, if yes it uses that instance but if not it uses an instance of `Console\Scheduling\CacheMutex`:

当 `Console\Scheduling\Schedule` 的实例被实例化时，laravel会检查 `Console\Scheduling\Mutex` 接口的实现是否绑定到容器，如果是，则使用该实例，如果不是，使用`Console\Scheduling\CacheMutex`实例：

```php
$this->mutex = $container->bound(Mutex::class)
                        ? $container->make(Mutex::class)
                        : $container->make(CacheMutex::class);
```

Now while the Schedule Manager is registering your events it'll pass an instance of the mutex:

现在，Schedule Manager正在注册你的事件，它会传递互斥的一个实例：

```php
$this->events[] = new Event($this->mutex, $command);
```

By default Laravel uses a cache-based mutex, but you can override that and implement your own mutex approach & bind it to the container.

默认情况下，Laravel使用基于缓存的互斥，但您可以覆盖它并实现自己的互斥方法并将其绑定到容器。

### The cache-based mutex

### 基于缓存的互斥

The `CacheMutex` class contains 3 simple methods, it uses the event mutex name as a cache key:

`CacheMutex` 类包含3个简单的方法，它使用事件互斥名作为缓存键：

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

### Mutex removal after task finishes

###任务完成后的互斥删除

As we've seen before, the manager registers an after-callback that removes the mutex after the task is done, for a task that runs a command on the OS that might be enough to ensure that the mutex is cleared, but for a callback task the script might die while executing the callback, so to prevent that an extra fallback was added in `Console\Scheduling\CallbackEvent::run()`:

如前所述，管理器注册一个在完成任务之后删除互斥的回调，对于在操作系统上运行命令的任务可能足以确保互斥被清除，但是对于回调 执行回调时脚本可能会死机，所以为了防止这种情况在 `Console\Scheduling\CallbackEvent::run()`中添加了一个额外的回退：

```php
register_shutdown_function(function () {
    $this->removeMutex();
});
```