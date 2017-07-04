There are several ways to push jobs into the queue:

有几种方法可以将作业推送到队列中：

```php
Queue::push(new InvoiceEmail($order));

Bus::dispatch(new InvoiceEmail($order));

dispatch(new InvoiceEmail($order));

(new InvoiceEmail($order))->dispatch();
```

As explained in a [previous dive](https://divinglaravel.com/queue-system/preparing-jobs-for-queue), calls on the `Queue` facade are calls on the queue driver your app uses, calling the `push` method for example is a call to the `push` method of the `Queue\DatabaseQueue` class in case you're using the database queue driver.

There are several useful methods you can use:

调用`Queue` facade是对应用程序使用的队列驱动的调用，如果你使用数据库队列驱动，调用`push`方法是调用`Queue\DatabaseQueue`类的`push`方法。

有几种有用的方法可以使用：

```php
// 将作业推送到特定的队列
Queue::pushOn('emails', new InvoiceEmail($order));

// 在给定的秒数之后推送作业
Queue::later(60, new InvoiceEmail($order));

// 延迟后将作业推送到特定的队列
Queue::laterOn('emails', 60, new InvoiceEmail($order));

// 推送多个作业
Queue::bulk([
    new InvoiceEmail($order),
    new ThankYouEmail($order)
]);

// 推送特定队列上的多个作业
Queue::bulk([
    new InvoiceEmail($order),
    new ThankYouEmail($order)
], null, 'emails');
```

After calling any of these methods, the selected queue driver will store the given information in a storage space for workers to pick up on demand.

调用这些方法之后，所选择的队列驱动会将给定的信息存储在存储空间中，供workers按需获取。

#### Using the command bus

#### 使用命令总线

Dispatching jobs to queue using the command bus gives you extra control; you can set the selected `connection`, `queue`, and `delay` from within your job class, decide if the command should be queued or run instantly, send the job through a pipeline before running it, actually you can even handle the whole queueing process from within your job class.

The `Bus` facade proxies to the `Contracts\Bus\Dispatcher` container alias, this alias is resolved into an instance of `Bus\Dispatcher` inside `Bus\BusServiceProvider`:

使用命令总线调度作业进行排队可以给你额外控制权; 您可以从作业类中设置选定的`connection`, `queue`, and `delay` 来决定命令是否应该排队或立即运行，在运行之前通过管道发送作业，实际上你甚至可以从你的作业类中处理整个队列过程。

`Bug` facade代理到` Contracts\Bus\Dispatcher` 容器别名，此别名解析为`Bus\Dispatcher`内的`Bus\BusServiceProvider`的一个实例：

```
$this->app->singleton(Dispatcher::class, function ($app) {
    return new Dispatcher($app, function ($connection = null) use ($app) {
        return $app[QueueFactoryContract::class]->connection($connection);
    });
});
```

所以`Bus::dispatch()` 调用的 `dispatch()` 方法是 `Bus\Dispatcher` 类的:

```
public function dispatch($command)
{
    if ($this->queueResolver && $this->commandShouldBeQueued($command)) {
        return $this->dispatchToQueue($command);
    } else {
        return $this->dispatchNow($command);
    }
}
```

This method decides if the given job should be dispatched to queue or run instantly, the `commandShouldBeQueued()` method checks if the job class is an instance of `Contracts\Queue\ShouldQueue`, so using this method your job will only be queued in case you implement the `ShouldQueue` interface.

We're not going to look into `dispatchNow` in this dive, it'll be discussed in detail when we dive into how workers run jobs. For now let's look into how the Bus dispatches your job to queue:

该方法决定是否将给定的作业分派到队列或立即运行，`commandShouldBeQueued()` 方法检查作业类是否是 `Contracts\Queue\ShouldQueue`, 的实例，因此使用此方法，您的作业只有继承了`ShouldQueue`接口才会被放到队列中。

我们不会在这篇中深入`dispatchNow`，我们将在深入worker如何执行作业中详细讨论。 现在我们来看看总线如何调度你的工作队列：

```php
public function dispatchToQueue($command)
{
    $connection = isset($command->connection) ? $command->connection : null;

    $queue = call_user_func($this->queueResolver, $connection);

    if (! $queue instanceof Queue) {
        throw new RuntimeException('Queue resolver did not return a Queue implementation.');
    }

    if (method_exists($command, 'queue')) {
        return $command->queue($queue, $command);
    } else {
        return $this->pushCommandToQueue($queue, $command);
    }
}
```

First Laravel checks if a `connection` property is defined in your job class, using the property you can set which connection Laravel should send the queued job to, if no property was defined `null` will be used and in such case Laravel will use the default connection.

Using the connection value, Laravel uses a queueResolver closure to build the instance of the queue driver that should be used, this closure is set inside `Bus\BusServiceProvider` while registering the Dispatcher instance:

首先 Laravel会检查您的作业类中是否定义了`connection` 属性，使用这个属性可以设置Laravel应该将排队作业发送到哪个连接，如果未定义任何属性，将使用null属性，在这种情况下Laravel将使用默认连接。

通过设置的连接，Laravel使用一个queueResolver闭包来构建应该使用哪个队列驱动的实例，当注册调度器实例的时候这个闭包在`Bus\BusServiceProvider` 中被设置：

```php
function ($connection = null) use ($app) {
    return $app[Contracts\Queue\Factory::class]->connection($connection);
}
```

`Contracts\Queue\Factory` is an alias for `Queue\QueueManager`, so in other words this closure returns an instance of QueueManager and sets the desired connection for the manager to know which driver to use.

Finally the `dispatchToQueue` method checks if the job class has a `queue` method, if that's the case the dispatcher will just call this method giving you full control over how the job should be queued, you can select the queue, assign delay, set maximum retries, timeout, etc...

In case no `queue` method was found, a call to `pushCommandToQueue()` calls the proper `push`method on the selected queue driver:

`Contracts\Queue\Factory` 是`Queue\QueueManager`的别名，换句话说，该闭包返回一个`QueueManager`实例，并为`manager`设置所使用的队列驱动需要的连接。

最后，`dispatchToQueue`方法检查作业类是否具有`queue`方法，如果调度器调用此方法，可以完全控制作业排队的方式，您可以选择队列，分配延迟，设置最大重试次数， 超时等

如果没有找到 `queue` 方法，对 `pushCommandToQueue()` 的调用将调用所选队列驱动上的`push`方法：

```php
protected function pushCommandToQueue($queue, $command)
{
    if (isset($command->queue, $command->delay)) {
        return $queue->laterOn($command->queue, $command->delay, $command);
    }

    if (isset($command->queue)) {
        return $queue->pushOn($command->queue, $command);
    }

    if (isset($command->delay)) {
        return $queue->later($command->delay, $command);
    }

    return $queue->push($command);
}
```

The dispatcher checks for `queue` and `delay` properties in your Job class & picks the appropriate queue method based on that.

调度器检查Job类中的 `queue` 和 `delay` ，并根据此选择适当的队列方法。

#### So I can set the queue, delay, and connection inside the job class?

#### 所以我可以设置工作类中的队列，延迟和连接？

Yes, you can also set a `tries` and `timeout` properties and the queue driver will use these values as well, here's how your job class might look like:

是的，您还可以设置一个`tries` 和 `timeout` 属性，队列驱动也将使用这些值，以下工作类示例：

```php
class SendInvoiceEmail{
    public $connection = 'default';

    public $queue = 'emails';

    public $delay = 60;

    public $tries = 3;

    public $timeout = 20;
}
```

#### Setting job configuration on the fly

#### 即时设置作业配置

Using the `dispatch()` global helper you can do something like this:

使用 `dispatch()` 全局帮助方法，您可以执行以下操作：

```php
dispatch(new InvoiceEmail($order))
        ->onConnection('default')
        ->onQueue('emails')
        ->delay(60);
```

This only works if you use the `Bus\Queueable` trait in your job class, this trait contains several methods that you may use to set some properties on the job class before dispatching it, for example:

这只有在您在作业类中使用 `Bus\Queueable` trait时才有效，此trait包含几种方法，您可以在分发作业类之前在作业类上设置一些属性，例如：

```php
public function onQueue($queue)
{
    $this->queue = $queue;

    return $this;
}
```

#### But in your example we call the methods on the return of dispatch()!

#### 但是在你的例子中，我们调用dispatch()的返回方法！

Here's the trick:

这是诀窍：

```php
function dispatch($job)
{
    return new PendingDispatch($job);
}
```

This is the definition of the `dispatch()` helper in `Foundation/helpers.php`, it returns an instance of `Bus\PendingDispatch` and inside this class we have methods like this:

这是在`Foundation/helpers.php`中的`dispatch()`帮助方法的定义，它返回一个`Bus\PendingDispatch` 的实例，并且在这个类中，我们有这样的方法：

```php
public function onQueue($queue)
{
    $this->job->onQueue($queue);

    return $this;
}
```

So when we do `dispatch(new JobClass())->onQueue('default')`, the `onQueue` method of `PendingDispatch` will call the `onQueue` method on the job class, as we mentioned earlier job classes need to use the `Queueable` trait for all this to work.

所以当我们执行 `dispatch(new JobClass())->onQueue('default')`, 时，`PendingDispatch` 的 `onQueue` 方法将调用job类上的 `onQueue` 方法，如前所述，作业类需要使用所有这些的 `Queueable` trait来工作。

#### Then where's the part where the Dispatcher::dispatch method is called?

#### 那么调用Dispatcher::dispatch方法的那部分是哪里？

Once you do `dispatch(new JobClass())->onQueue('default')` you'll have the job instance ready for dispatching, the actual work happens inside `PendingDispatch::__destruct()`:

一旦你执行了 `dispatch(new JobClass())->onQueue('default')`，你将让作业实例准备好进行调度，实际的工作发生在 `PendingDispatch::__destruct()`中：

```php
public function __destruct()
{
    app(Dispatcher::class)->dispatch($this->job);
}
```

This method, when called, will resolve an instance of `Dispatcher` from the container and call the `dispatch()` method on it. A `__destruct()` is a PHP magic method that's called when all references to the object no longer exist or when the script terminates, and since we don't store a reference to the `PendingDispatch` instance anywhere the `__destruct` method will be called immediately:

调用此方法时，将从容器中解析 `Dispatcher` 的一个实例，然后调用它的`dispatch()`方法。 __destruct（）是一种PHP魔术方法，当对对象的所有引用不再存在或脚本终止时，都会调用，因为我们不会立即在__ `__destruct`方法中存储对`PendingDispatch` 实例的引用，

```php
// Here the destructor will be called rightaway
dispatch(new JobClass())->onQueue('default');

// 如果我们调用unset($temporaryVariable)，那么析构函数将被调用
// 或脚本完成执行时。
$temporaryVariable = dispatch(new JobClass())->onQueue('default');
```

#### Using the Dispatchable trait

#### 使用可调度的特征

You can use the `Bus\Dispatchable` trait on your job class to be able to dispatch your jobs like this:

您可以使用工作类上的 `Bus\Dispatchable` trait来调度您的工作，如下所示：

```php
(new InvoiceEmail($order))->dispatch();
```

Here's how the dispatch method of `Dispatchable` looks like:

调度方法 `Dispatchable`看起来像这样：

```php
public static function dispatch()
{
    return new PendingDispatch(new static(...func_get_args()));
}
```

As you can see it uses an instance of `PendingDispatch`, that means we can do something like:

正如你可以看到它使用一个 `PendingDispatch`的实例，这意味着我们可以做一些像这样的事：

```php
(new InvoiceEmail($order))->dispatch()->onQueue('emails');
```