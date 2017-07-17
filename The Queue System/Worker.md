Now that we learnt how Laravel pushes jobs into different queues, let's perform a dive into how workers run your jobs. First I'd like to define workers as a simple PHP process that runs in the background with the purpose of extracting jobs from a storage space and run them with respect to several configuration options.

现在，我们知道了Laravel如何将作业推到不同的队列中，让我们来深入了解workers如何运作你的作业。 首先，我将workers定义为一个在后台运行的简单PHP进程，目的是从存储空间中提取作业并针对多个配置选项运行它们。

```php
php artisan queue:work
```

Running this command will instruct Laravel to create an instance of your application and start executing jobs, this instance will stay alive indefinitely which means the action of starting your Laravel application happens only once when the command was run & the same instance will be used to execute your jobs, that means the following:

- You save server resources by avoiding booting up the whole app on every job.
- You have to manually restart the worker to reflect any code change you made in your application.

运行此命令将指示Laravel创建应用程序的一个实例并开始执行作业，这个实例将一直存活着，启动Laravel应用程序的操作只在运行命令时发生一次，同一个实例将被用于执行你的作业，这意味着：

- 避免在每个作业上启动整个应用程序来节省服务器资源。
- 在应用程序中所做的任何代码更改后必须手动重启worker。

你也可以这样运行:

```php
php artisan queue:work --once
```

This will start an instance of the application, process a single job, and then kill the script.

这将启动应用程序的一个实例，处理单个作业，然后干掉脚本。

```php
php artisan queue:listen
```

The `queue:listen` command simply runs the `queue:work --once` command inside an infinite loop, this will cause the following:

- An instance of the app is booted up on every loop.
- The assigned worker will pick a single job and execute it.
- The worker process will be killed.

`queue:listen` 命令相当于无限循环地运行 `queue:work --once` 命令，这将导致以下问题：

- 每个循环都会启动一个应用程序实例。
- 分配的worker将选择一个工作并执行。
- worker进程将被干掉。

Using `queue:listen` ensures that a new instance of the app is created for every job, that means you don't have to manually restart the worker in case you made changes to your code, but also means more server resources will be consumed.

使用 `queue:listen` 确保为每个作业创建一个新的应用程序实例，这意味着代码更改以后不必手动重启worker，同时也意味着将消耗更多的服务器资源。

# The queue:work command

# queue:work 命令

Let's take a look at the `handle()` method of the `Queue\Console\WorkCommand` class, it's the method that'll be executed when you run `php artisan queue:work`:

我们来看看 `Queue\Console\WorkCommand` 类的 `handle()` 方法，这是当你运行 `php artisan queue:work` 时会执行的方法：

```php
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

首先，我们检查应用程序是否处于维护模式，并使用 `--once` 选项，在这种情况下，我们希望脚本正常运行，因此我们不执行任何作业，我们只需要在完全杀死脚本前让worker在一段时间内休眠。

Here's how the `sleep()` method of `Queue\Worker` looks like:

`Queue\Worker` 的 `sleep()` 方法看起来像这样：

```php
public function sleep($seconds)
{
    sleep($seconds);
}
```

#### Why don't we just return null in the handle() method to kill the script?

#### 为什么我们不能在 handle() 方法中返回null来终止脚本？

As we said earlier the `queue:listen` command runs the `WorkCommand` inside a loop:

如前所述， `queue:listen` 命令在循环中运行 `WorkCommand` ：

```php
while (true) {
     // This process simply calls 'php artisan queue:work --once'
    $this->runProcess($process, $options->memory);
}
```

If the app is in maintenance mode and the `WorkCommand` terminated immediately this will cause the loop to end and a next one starts in a very short time, it's better that we cause some delay in that case instead of consuming the server resources by creating tons of application instances that we won't really use.

如果应用程序处于维护模式，并且 `WorkCommand` 立即终止，这将导致循环结束，下一个在很短的时间内启动，最好在这种情况下导致一些延迟，而不是通过创建我们不会真正使用的大量应用程序实例。

## Listening for events

## 监听事件

Inside the `handle()` method we call the `listenForEvents()` method:

在 `handle()` 方法里面我们调用 `listenForEvents()` 方法：

```php
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

在这个方法中我们会监听几个事件，这样我们可以在每次作业处理中，处理完或处理失败时向用户打印一些信息。

## Logging failed jobs

## 记录失败作业

In case a job fails, the `logFailedJob()` method is called:

一旦作业失败 `logFailedJob()` 方法会被调用

```php
$this->laravel['queue.failer']->log(
    $event->connectionName, $event->job->getQueue(),
    $event->job->getRawBody(), $event->exception
);
```

The `queue.failer` container alias is registered in `Queue\QueueServiceProvider::registerFailedJobServices()`:

`queue.failer` 容器别名在 `Queue\QueueServiceProvider::registerFailedJobServices()` 中注册:

```php
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

如果配置了  `queue.failed` ，则将使用数据库队列失败，并将有关失败作业的信息简单地存储在数据库表中的：

```php
$this->getTable()->insertGetId(compact(
    'connection', 'queue', 'payload', 'exception', 'failed_at'
));
```

## Running the worker

## 运行worker

To run the worker we need to collect two pieces of information:

- The connection this worker will be pulling jobs from
- The queue the worker will use to find jobs

要运行worker，我们需要收集两条信息：

- worker的连接信息从作业中提取
- worker找到作业的队列

You can provide a `--connection=default` option to the `queue:work` command, but in case you didn't the default collection defined in the `queue.default` configuration value will be used.

如果没有使用 `queue.default` 配置定义的默认连接。您可以为 `queue:work` 命令提供 `--connection=default` 选项。

Same for the queue, you can provide a `--queue=emails` option or the `queue` option set in your selected connection configuration will be used.

Once all this is done, the `WorkCommand::handle()` method runs `runWorker()`:

队列也是一样，您可以提供一个 `--queue=emails` 选项，或选择连接配置中的 `queue` 选项。一旦这一切完成， `WorkCommand::handle()` 方法会运行 `runWorker（）`：

```php
protected function runWorker($connection, $queue)
{
    $this->worker->setCache($this->laravel['cache']->driver());

    return $this->worker->{$this->option('once') ? 'runNextJob' : 'daemon'}(
        $connection, $queue, $this->gatherWorkerOptions()
    );
}
```

The worker class property is set while the command is being constructed:

在worker类属性在命令构造后设置：

```php
public function __construct(Worker $worker)
{
    parent::__construct();

    $this->worker = $worker;
}
```

The container resolves an instance of `Queue\Worker`, inside `runWorker()` we set the cache driver the worker will use, we also decide what method we'll call to based on the `--once` command option.

容器解析 `Queue\Worker` 实例，在`runWorker()`中我们设置了worker将使用的缓存驱动，我们也根据`--once` 命令来决定我们调用什么方法。

In case the `--once` option is used we'll just call `runNextJob` to run the next available job and then the script dies. Otherwise we'll call the `daemon` method which will keep the process alive processing jobs all the time.

如果使用 `--once` 选项，我们只需调用 `runNextJob` 来运行下一个可用的作业，然后脚本就会终止。 否则，我们将调用 `daemon` 方法来始终保持进程处理作业。

While starting the worker we gather the command options given by the user using the `gatherWorkerOptions()` method, we later provide these options the worker `runNextJob` or `daemon` methods.

在开始工作时，我们使用 `gatherWorkerOptions()` 方法收集用户给出的命令选项，我们稍后会提供这些选项，这个工具是 `runNextJob` 或 `daemon` 方法。

```php
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

## 守护进程

Let's take a look at the `Worker::daemon()` method, first line in this method calls the `listenForSignals()` method:

让我看看 `Worker::daemon()` 方法，这个方法的第一行调用了 `Worker::daemon()` 方法

```php
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

这种方法使用PHP7.1的信号处理， `supportsAsyncSignals()` 方法检查我们是否在PHP7.1上，并加载` pcntl` 扩展名。

`pcntl_async_signals()` is later called to enable signal handling, then we register handlers for multiple signals:

- `SIGTERM` is raised when the script is instructed to shutdown.
- `SIGUSR2` is a user-defined signal Laravel uses to indicate that the script should pause.
- `SIGCONT` is raised when a paused script should proceed.

之后`pcntl_async_signals()` 被调用来启用信号处理，然后我们为多个信号注册处理程序：

- 当脚本被指示关闭时，会引发`SIGTERM`。
- `SIGUSR2`是用户定义的信号，Laravel用来表示脚本应该暂停。
- 当暂停的脚本继续进行时，会引发`SIGCONT`。

These signals are sent from a Process Monitor such as [Supervisor](http://supervisord.org/) to communicate with our script.

这些信号从Process Monitor（如 [Supervisor](http://supervisord.org/) ）发送并与我们的脚本进行通信。

Second line in the `Worker::daemon()` method fetches the timestamp of last queue restart, this value is stored in the cache when we call the `queue:restart`, later on we'll check if the timestamp of last restart doesn't match which indicates that the worker should restart, more on that later.

 `Worker::daemon()` 方法中的第二行读取最后一个队列重新启动的时间戳，当我们调用`queue:restart` 时该值存储在缓存中，稍后我们将检查是否和上次重新启动的时间戳不符合，来指示worker在之后多次重启。

Finally the method starts a loop where we'll do the rest of the work of fetching jobs, running them, and do several actions on the worker process.

最后，该方法启动一个循环，在这个循环中，我们将完成其余获取作业的worker，运行它们，并对worker进程执行多个操作。

```php
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

### 确定worker是否应该处理作业

Calling `daemonShouldRun()` we check for the following cases:

- Application is not in maintenance mode
- Worker is not paused
- y

调用 `daemonShouldRun()` 检查以下情况：

- 应用程序不处于维护模式
- Worker没有暂停
- 没有事件监听器阻止循环继续

If app in maintenance mode you can still process jobs if your worker run with the `--force` option:

如果应用程序在维护模式下，worker使用`--force`选项仍然可以处理作业：

```php
php artisan queue:work --force
```

One of the conditions determining if the worker should continue is the following:

确定worker是否应该继续的条件之一是：


```php
$this->events->until(new Events\Looping($connectionName, $queue)) === false)
```

This line fires a `Queue\Event\Looping` event and checks if any of the listeners return false in its `handle()` method, using this fact you can occasionally force your workers to stop processing jobs temporarily.

In case the worker should pause, the `pauseWorker()` method is called:

这行触发 `Queue\Event\Looping` 事件，并检查是否有任何监听器在 `handle()` 方法中返回false，这种情况下你可以强制您的workers暂时停止处理作业。

如果worker应该暂停，则调用 `pauseWorker()` 方法：


```php
protected function pauseWorker(WorkerOptions $options, $lastRestart)
{
    $this->sleep($options->sleep > 0 ? $options->sleep : 1);

    $this->stopIfNecessary($options, $lastRestart);
}
```

 the `sleep` method and passes the `--sleep` option given to the console command:
This method calls

`sleep` 方法并传递给控制台命令的 `--sleep` 选项，这个方法调用

```php
public function sleep($seconds)
{
    sleep($seconds);
}
```

After the script sleeps for a while, we check if the worker should quit and kill the script in that case, we'll look into the `stopIfNecessary` method later, in case the script shouldn't be killed we'll just call `continue;` to start a new loop:

脚本休眠了一段时间后，我们检查worker是否应该在这种情况下退出并杀死脚本，稍后我们看一下`stopIfNecessary` 方法，以防脚本不能被杀死，我们只需调用 `continue;` 开始一个新的循环：

```php
if (! $this->daemonShouldRun($options, $connectionName, $queue)) {
    $this->pauseWorker($options, $lastRestart);

    continue;
}
```

### Retrieving a job to run

### Retrieving 要运行的作业

```php
$job = $this->getNextJob(
    $this->manager->connection($connectionName), $queue
);
```

The `getNextJob()` method accepts an instance of the desired queue connection and the queue we should fetch jobs from:

`getNextJob()` 方法接受一个队列连接的实例，我们从队列中获取作业

```php
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

我们简单地循环给定的队列，使用选择的队列连接从存储空间（数据库，redis，sqs，...）获取作业并返回该作业。

To retrieve a job from storage we query for the oldest job that meets the following conditions:

- Pushed to the `queue` we're trying to find jobs within
- Not reserved by another worker
- Available to be run at the given time, some jobs are delayed to run in the future
- We also get jobs that are reserved for a long time they became frozen so we retry them

要从存储中retrieve作业，我们查询满足以下条件的最旧作业：

- 推送到 `queue` ，我们试图从中找到作业
- 没有被其他worker reserved
- 可以在给定的时间内运行，有些作业在将来被推迟运行
- 我们也取到了很久以来被冻结的作业并重试

Once we find a job that meets this criteria we mark this job as reserved so that other workers don't pick it up, we also increment the number of attempts of the job for monitoring.

一旦我们找到符合这一标准的作业，我们将这个作业标记为reserved，以便其他workers获取到，我们还会增加作业监控次数。

### Monitoring jobs timeout

### 监控作业超时

After the next job is retrieved, we call the `registerTimeoutHandler()` method:

下一个作业被retrieved之后，我们调用 `registerTimeoutHandler()` 方法：

```php
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

再次，如果 `pcntl` 扩展被加载，我们将注册一个信号处理程序干掉worker进程如果该作业超时的话，在配置了超时之后我们使用 `pcntl_alarm()` 来发送一个 `SIGALRM` 信号。

If the job took longer than the timeout value the handler will kill the script, if not the job will pass and the next loop will set a new alarm overriding the first one since a single alarm can be present in the process.

如果作业所花费的时间超过了超时值，处理程序将会终止该脚本，如果不是该作业将通过，并且下一个循环将设置一个新的报警覆盖第一个报警，因为进程中可能存在单个报警。

Jobs timeout only work on PHP7.1 or above, it also doesn't work on Windows ¯\_(ツ)_/¯

作业只在PHP7.1以上起效,在window上也无效 ¯\_(ツ)_/¯

### Processing a job

### 处理作业

The `runJob()` method calls `process()`:

`runJob()` 方法调用 `process()`:

```php
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

Here `raiseBeforeJobEvent()` fires the `Queue\Events\JobProcessing` event, and `raiseAfterJobEvent()` fires the `Queue\Events\JobProcessed` event.`markJobAsFailedIfAlreadyExceedsMaxAttempts()` checks if the process already reached the maximum attempts and mark the job as failed in that case:

`raiseBeforeJobEvent()` 触发  `Queue\Events\JobProcessing` 事件,  `raiseAfterJobEvent()` 触发 `Queue\Events\JobProcessed` 事件。 `markJobAsFailedIfAlreadyExceedsMaxAttempts()` 检查进程是否达到最大尝试次数，并将该作业标记为失败：




```php
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

否则我们在作业对象上调用 `fire()` 方法来运行作业。

#### Where did we get this job object?

#### 从哪里获取作业对象

The `getNextJob()` method returns an instance of `Contracts\Queue\Job`, depends on the queue driver we use the respective Job instance will be used, for example `Queue\Jobs\DatabaseJob` in case the Database queue driver is selected.

`getNextJob()` 方法返回一个 `Contracts\Queue\Job` 的实例，这取决于我们使用相应的Job实例的队列驱动程序，例如如果数据库队列驱动则选择 `Queue\Jobs\DatabaseJob` 。

### End of loop

### 循环结束

At the end of the loop we call `stopIfNecessary()` to check if we should kill the process before the next loop starts:

在循环结束时，我们调用 `stopIfNecessary()` 来检查在下一个循环开始之前是否应该停止进程：

```php
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

`shouldQuit` 属性在两种情况下设置，首先`listenForSignals()` 内部的作为 `SIGTERM` 信号处理程序，其次在 `stopWorkerIfLostConnection()` 中

```php
protected function stopWorkerIfLostConnection($e)
{
    if ($this->causedByLostConnection($e)) {
        $this->shouldQuit = true;
    }
}
```

This method is called in several try...catch statements while retrieving and processing jobs to ensure that the worker should die so that our Process Control may start a new one with a fresh database connection.

在retrieving和处理作业时，会在几个try ... catch语句中调用此方法，以确保worker应该处于被干掉的状态，以便我们的Process Control可能会启动一个新的数据库连接。

The `causedByLostConnection()` method can be found in the `Database\DetectsLostConnections` trait.

`causedByLostConnection()` 方法可以在 `Database\DetectsLostConnections` trait中找到。

`memoryExceeded()` checks if memory usage exceeded the current set memory limit, you can set the limit using the `--memory` option on the `queue:work` command.

`memoryExceeded()` 检查内存使用情况是否超过当前设置的内存限制，您可以使用 `--memory` 选项设置限制。

Finally `queueShouldRestart()` compares the current timestamp of a restart signal and if it doesn't match the time we stored while starting the worker process this means that a new restart signal was sent during the loop, in that case we kill the process so that it can be restarted by the Process Control later.

最后`queueShouldRestart()` 比较重新启动信号的当前时间戳，如果它与启动工作进程时存储的时间不匹配，这意味着在循环中发送了一个新的重启信号，在这种情况下，我们终止进程以便稍后可以由Process Control重新启动它。