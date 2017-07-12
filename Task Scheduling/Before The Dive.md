Imagine this scenario, as a developer of a large SaaS you're tasked with finding a way to select 10 random customers every minute during the weekend and offer them a discounted upgrade, the job for sending the discount can be pretty easy but we need a way to run it every minute, for that let me share a brief introduction about CRON for those who're not familiar with it.

想象这种情况，作为一个大型SaaS的开发者，您需要找到一种在周末每分钟选择10个随机客户的方式，并提供折扣升级，发送折扣的工作可能非常简单，但我们需要每分钟运行一次，为此我分享一些CRON的简要介绍给还不熟悉人。

# CRON

CRON is a daemon that lives inside your linux server, it's not awake most of the time but every minute it'll open its eyes and see if it's time to run any specific task that was given to it, you communicate with that daemon using crontab files, in most common setups that file can be located at `/etc/crontab`, here's how a crontab file might look like:

CRON是一个守护进程，它驻留在你的linux服务器中，大部分时间都没有唤醒，但是每一分钟它都会睁开双眼，看看是否运行任何给定的任务，你使用crontab文件与该守护进程通信，在大多数常见的设置文件可以位于`/etc/crontab`，crontab文件可能看起来像这样：

```
0 0 1 * * /home/full-backup
0 0 * * * /home/partial-backup
30 5 10 * * /home/check-subscriptions
```

In the crontab file each line represents a scheduled job, and each job definition contains two parts:

1. The ***** part represents the timer for that job to run.
2. The second part is the command that should run

在crontab文件中，每行表示一个计划任务作业，每个作业定义包含两部分： 

1. *****部分代表该作业运行的计时器。   
2. 第二部分是应运行的命令   

### CRON Timing Syntax

### CRON时序语法

The 5 asterisks represent the following in order:

1. Minute of an hour
2. Hour of a day
3. Day of a month
4. Month of a year
5. Day of a week

5个星号按顺序排列如下：

1. 一小时内的分钟
2. 一天内的小时
3. 一个月内的日期
4. 一年内的月份
5. 一周的内的天

示例：

- `0 0 1 * *` in the first example indicates that the job should run on every month, at the first day of the month, at 12 AM, at the first minute of the hour. Or simply it should run every 1st day of the month at 12:00 AM.
- `0 0 1 * *` 在第一个例子中，表示该工作应在每月，每个月的第一个天，上午12点，每小时第一分钟运行。 或者简单地说，它应该在每月的第一天上午12:00运行。
- `0 * * * *` in the second example indicates that the job should run every hour.
- `0 * * * *` 在第二个例子中，表示该工作应该每小时运行一次。
- `30 5 10 * *` indicates that the job should run on the 10th of every month at 5:30 AM
- `30 5 10 * *` 表示该工作应该在每个月10日上午5:30运行


Here are a few other examples:

- `* * * * 3` indicates that the job should run every minute on Wednesdays.
- `* * * * 1-5` indicates that the job should run every minute Monday to Friday.
- `0 1,15 * * *` indicates that the job should run twice a day at 1AM and 3PM.
- `*/10 * * * *` indicates that the job should run every 10 minutes.

这里还有一些其他的示例:

- `* * * * 3` indicates that the job should run every minute on Wednesdays.
- `* * * * 3` 表示工作应该在星期三每分钟运行一次。
- `* * * * 1-5` indicates that the job should run every minute Monday to Friday.
- `* * * * 1-5` 表示该工作应该每周一至周五运行。
- `0 1,15 * * *` indicates that the job should run twice a day at 1AM and 3PM.
- `0 1,15 * * *` 表示该工作应该每天在凌晨1点和3点运行两次。
- `*/10 * * * *` indicates that the job should run every 10 minutes.
- `*/10 * * * *` 表示该工作应该每10分钟运行一次。

#### So we register a cron task for our job?

#### 所以我们为我们的工作注册一个cron任务？

Yeah we can simply register this in our crontab file:
是的，我们可以在我们的crontab文件中注册：

```
 * * * * php /home/divingLaravel/artisan send:offer
```

This command will inform the CRON daemon to run the `php artisan send:offer` artisan command every minute, pretty easy right? But it then gets confusing when we want to run the command every minute only on Thursdays and Tuesdays, or on specific days of the month, having to remember the syntax for cron jobs is not an easy job and also having to update the crontab file every time you want to add a new job or update the schedule can be pretty time consuming sometimes, so a few releases back Laravel added some interesting feature that provides an easy to remember syntax for scheduling tasks:

该命令将通知CRON守护程序每分钟运行 `php artisan send：offer` artisan命令，是不是很容易？ 但是，当我们想要在星期四和星期二或每个特定日子里每分钟运行命令时会感到困惑，记住cron作业的语法不是一件容易的事，而且还需要更新crontab文件，你想添加一个新的工作或更新的时间表可能是相当耗时的时间，所以几个版本发布后Laravel添加了一些有趣的功能，为调度任务提供了一个容易记住的语法：

```php
$schedule->command('send:offer')
          ->everyFiveMinutes()
          ->wednesdays();
```

You only have to register one cron jobs in your crontab and laravel takes care of the rest under the hood:
你只需要在你的crontab中注册一个cron工作，laravel会处理剩下的事：

```php
* * * * * php /divingLaravel/artisan schedule:run >> /dev/null 2>&1
```

You may define your scheduled commands inside the schedule method of your `App\Console\Kernel` class:

您可以在`App\Console\Kernel`类的schedule方法中定义预定的命令：

```php
protected function schedule(Schedule $schedule)
{
    $schedule->command('send:offer')
          ->everyFiveMinutes()
          ->wednesdays();
}
```

If you'd like more information about the different Timer options, take a look at the [official documentation](https://laravel.com/docs/5.4/scheduling#schedule-frequency-options).

如果您想了解有关不同计时器选项的更多信息，请查看 [官方文档](https://laravel.com/docs/5.4/scheduling#schedule-frequency-options)。

While the Console Kernel instance is being instantiated, Laravel registers a listener to the Kernel's `booted` event that binds the Scheduler to the container and calls the schedule() method of the kernel:

当Console Kernel被实例化时，Laravel向内核的`booted`事件注册一个侦听器，该事件将Scheduler绑定到容器并调用kernel的schedule()方法：

```php
// in Illuminate\Foundation\Console\Kernel

public function __construct(Application $app, Dispatcher $events)
{
    $this->app->booted(function () {
        $this->defineConsoleSchedule();
    });
}

protected function defineConsoleSchedule()
{
     // Register the Scheduler in the Container
    $this->app->instance(
        Schedule::class, $schedule = new Schedule($this->app[Cache::class])
    );

     // Call the schedule() method that we override in our App\Console\Kernel
    $this->schedule($schedule);
}
```

This `booted` event is fired once the console kernel finishes the bootstrapping sequence defined in the Kernel class.

一旦console kernel完成Kernel类中定义的引导顺序，这个`booted`事件就被触发。

Inside the handle() method of the Kernel, Laravel checks if Foundation\Application was booted before, and if not it calls the bootstrapWith() method of the Application and passes the bootstrappers array defined in the console Kernel.

在Kernel的handle()方法中，Laravel会检查`Foundation\Application`是否已启动，如果不是调用应用程序的bootstrapWith()方法，并传递在console Kernel定义的引导程序数组。


#### Simply put:

#### 简单地说:

When the CRON daemon calls the `php artisan schedule:run` command every minute, the Console Kernel will be booted up and the jobs you defined inside your `App\Console\Kernel::schedule()` method will be registered into the scheduler.

当CRON守护程序每分钟都调用`php artisan schedule：run`命令时，控制台Console Kernel将被启动，您在`App\Console\Kernel::schedule()`方法中定义的作业将被注册到调度程序。

The `schedule()` method takes an instance of `Illuminate\Console\Scheduling\Schedule` as the only argument, this is the schedule manager used to record the jobs you give it and decides what should run every time the CRON daemon pings it.

`schedule()`方法调用`Illuminate\Console\Scheduling\Schedule`的实例作为唯一的参数，这是用于记录您提供的作业的计划任务管理器，并决定每次CRON守护进程应该运行什么。
