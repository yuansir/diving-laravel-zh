Every entry you add is converted into an instance of `Illuminate\Console\Scheduling\Event` and stored in an `$events` class property of the Scheduler, an Event object consists of the following:

你添加的每个记录都将转换为 `Illuminate\Console\Scheduling\Event` 的实例，并存储在Scheduler的 `$events` 类属性中，Event对象由以下内容组成：

- Command to run
- CRON Expression
- Timezone to be used to evaluate the time
- Operating System User the command should run as
- The list of Environments the command should run under
- Maintenance mode configuration
- Event Overlapping configuration
- Command Foreground/Background running configuration
- A list of checks to decide if the command should run or not
- Configuration on how to handle the output
- Callbacks to run after the command runs
- Callbacks to run before the command runs
- Description for the command
- A unique Mutex for the command

- 命令运行
- CRON表达式
- 用于评估时间的时区
- 操作系统运行该命令的用户
- 命令应该运行的环境列表
- 维护模式配置
- 事件重叠配置
- 命令前台/后台运行配置
- 用于决定该命令是否运行的检查列表
- 如何处理输出的配置
- 命令运行后运行的回调
- 在命令运行前运行的回调
- 命令说明
- 命令的唯一Mutex

The command to run could be one of the following:

命令可能像下面一种方式运行：

- A callback
- A command to run on the operating system
- An artisan command
- A job to be dispatched

- 回调
- 在操作系统上运行的命令
- artisan命令
- 被调度的作业

### Using a callback

### 使用回调

In case of a callback, the `Container::call()` method is used to run the value we pass which means we can pass a callable or a string representing a method on a class:

在回调的情况下，`Container::call()` 方法用于运行我们传递的值，这意味着我们可以传递一个可以调用或表示方法的字符串：

```php
protected function schedule(Schedule $schedule)
{
    $schedule->call(function () {
        DB::table('recent_users')->delete();
    })->daily();
}
```

Or:

```php
protected function schedule(Schedule $schedule)
{
    $schedule->call('MetricsRepository@cleanRecentUsers')->daily();
}
```

### Passing a command for the operating system

### 调用操作系统的命令

If you would like to pass a command for the operating system to run you can use `exec()`:

如果要运行操作系统的命令，可以使用 `exec()`：

```php
$schedule->exec('php /home/sendmail.php --user=10 --attachInvoice')->monthly();
```

You can also pass the parameters as an array:

您还可以将数组作为参数：

```php
$schedule->exec('php /home/sendmail.php', [
    '--user=10',
    '--subject' => 'Reminder',
    '--attachInvoice'
])->monthly();
```

### Passing an artisan command

### 调用一个artisan命令

```php
$schedule->command('mail:send --user=10')->monthly();
```

You can also pass the class name:

你也可以传一个类名

```php
$schedule->command('App\Console\Commands\EmailCommand', ['user' => 10])->monthly();
```

The values you pass are converted under the hood to an actual shell command and passed to `exec()` to run it on the operating system.

你传递的值将转换为实际的shell命令，并传递给 `exec()` 在操作系统上运行。

### Dispatching a Job

### 调度一个作业

You may dispatch a job to queue using the Job class name or an actual object:

您可以使用Job类名称或实际对象将作业分发到队列中：

```php
$schedule->job('App\Jobs\SendOffer')->monthly();

$schedule->job(new SendOffer(10))->monthly();
```

Under the hood Laravel will create a callback that calls the `dispatch()` helper method to dispatch your command.

Laravel会创建一个回调函数，调用 `dispatch()` 辅助方法来分发你的命令。

So the two actual methods of creating an event here is by calling `exec()` or `call()`, the first one submits an instance of `Illuminate\Console\Scheduling\Event` and the latter submits `Illuminate\Console\Scheduling\CallbackEvent` which has some special handling.

所以在这里创建一个事件的两个实际方法是通过调用 `exec()` 或 `call()`，第一个提交一个 `Illuminate\Console\Scheduling\Event` 的实例，后者提交  `Illuminate\Console\Scheduling\CallbackEvent`来做一些特殊处理。

## Building the cron expression

## 创建cron表达式

Using the timing method of the Scheduled Event, laravel builds a CRON expression for that event under the hood, by default the expression is set to run the command every minute:

使用计划事件的计时方法，laravel会为该事件创建一个CRON表达式，默认情况下，表达式设置为每分钟运行一次命令：

```php
* * * * * *
```

But when you call `hourly()` for example the expression will be updated to:

但当你调用 `hourly()` 时表达式会更新成这样:

```php
0 * * * * *
```

If you call `dailyAt('13:30')` for example the expression will be updated to:

当你调用 `dailyAt('13:30')` 时表达式会更新成这样:

```php
30 13 * * * *
```

If you call `twiceDaily(5, 14)` for example the expression will be updated to:

当你调用 `twiceDaily(5, 14)` 时表达式会更新成这样:

```php
0 5,14 * * * *
```

A very smart abstraction layer that saves you tons of research to find the right cron expression, however you can pass your own expression if you want as well:

一个非常聪明的抽象层，可以节省大量的精力来找到正确的cron表达式，但是如果你只要你想你也可以传递你自己的表达式：

```php
$schedule->command('mail:send')->cron('0 * * * * *');
```

#### How about timezones?

#### 如何设置时区?

If you want the CRON expression to be evaluated with respect to a specific timezone you can do that using:

如果您希望CRON表达式针对特定时区，则可以使用以下方式进行：

```php
->timezone('Europe/London')
```

Under the hood Laravel checks the timezone value you set and update the `Carbon` date instance to reflect that.

Laravel检查您设置的时区值，并更新 `Carbon` 日期实例使其起作用。

#### So laravel checks if the command is due using the CRON expression?

#### 那么Laravel会用CRON表达式检查命令是否到期吗？

Exactly, Laravel uses the [mtdowling/cron-expression](https://github.com/mtdowling/cron-expression) library to determine if the command is due based on the current system time (with respect to the timezone we set).

恰恰相反，Laravel使用 [mtdowling/cron-expression](https://github.com/mtdowling/cron-expression) 库来确定命令是否基于当前系统时间（相对于我们设置的时区）。

## Adding Constraints on running the command

### Duration constraints

## 在运行命令时添加限制

### 持续时间限制

For example if you want the command to run daily but only between two specific dates:

例如，如果您希望命令每天运行，但只能在两个特定日期之间运行：

```php
->between('2017-05-27', '2017-06-26')->daily();
```

And if you want to prevent it from running during a specific period:

如果你想防止它在一段特定的时间内运行：

```php
->unlessBetween('2017-05-27', '2017-06-26')->daily();
```

### Environment constraints

### 环境限制

You can use the `environments()` method to pass the list of environments the command is allowed to run under:

您可以使用 `environments()` 设置传递命令允许运行的环境列表：

```php
->environments('staging', 'production');
```

### Maintenance Mode

### 维护模式

By default scheduled commands won't run when the application is in maintenance mode, however you can change that by using:

默认情况下，当应用程序处于维护模式时，调度的命令不会运行，但是您可以通过使用以下命令来更改：

```php
->evenInMaintenanceMode()
```

### OS User

### 系统用户

You can set the Operating System user that'll run the command using:

你可以设置那个操作系统用户来执行这个命令:

```php
->user('forge')
```

Under the hood Laravel will use `sudo -u forge` to set the user on the operating system.

Laravel将使用 `sudo -u forge` 设置在操作系统上运行的用户。

### Custom Constraints

### 自定义限制

You can define your own custom constraint using the `when()` and `skip()` methods:

您可以使用 `when()` 和 `skip()` 方法定义自定义约束：

```php
// Runs the command only when the user count is greater than 1000
->when(function(){
    return User::count() > 1000;
});

// Runs the command unless the user count is greater than 1000
->skip(function(){
    return User::count() > 1000;
});
```

## Before and After callbacks

## 之前和之后回调函数

Using the `before()` and `then()` methods you can register callbacks that'll run before or after the command finishes execution:

使用 `before()` 和 `then()` 方法可以注册在命令完成执行之前或之后运行的回调函数：

```php
->before(function(){
    Mail::to('myself@Mail.com', new CommandStarted());
})
->then(function(){
    Mail::to('myself@Mail.com', new CommandFinished());
});
```

You can also ping URLs or webhooks using the `pingBefore()` and `thenPing()` methods:

您还可以使用 `pingBefore()` and `thenPing()` 方法ping URL或webhooks：

```php
->ping('https://my-webhook.com/start')->thenPing('https://my-webhook.com/finish')
```

Using these commands laravel registers a before/after callbacks under the hood and uses Guzzle to send a `GET` HTTP request:

使用这些命令laravel在注册一个前/后回调，并使用Guzzle发送一个 `GET` HTTP请求：

```php
return $this->before(function () use ($url) {
    (new HttpClient)->get($url);
});
```

