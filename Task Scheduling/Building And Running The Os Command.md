When it's time to fire a scheduled event, Laravel's schedule manager calls the `run()` method on the `Illuminate\Console\Scheduling\Event` object representing that event, this happens inside the `Illuminate\Console\Scheduling\ScheduleRunCommand`.

在启动计划任务的事件的时候，Laravel的进度管理器在`Illuminate\Console\Scheduling\Event`对象上调用 `run()` 方法，表示该事件发生在 `Illuminate\Console\Scheduling\ScheduleRunCommand` 内。

This `run()` method builds the command syntax and runs it on the operating system using the Symfony Process component, but before building the command it first checks if the command should be running in the background, by default all commands run in the foreground unless you use the following method while scheduling your command:

这个 `run()` 方法构建命令语法，并使用Symfony Process组件在操作系统上运行它，但在构建命令之前，它首先检查该命令是否应该在后台运行，默认情况下所有命令都在前台运行 除非你使用以下方法来调度命令：

```php
->runInBackground()
```

#### When do I need to run a command in the background?

#### 什么时候我需要在后台运行命令?

Imagine if you have several tasks that should run at the same time, say every hour, with the default settings Laravel will instruct the OS to run the commands one by one:

假设如果您有几个任务应该同时运行，比如每个小时，Laravel默认设置会指示操作系统逐个运行命令：

```php
~ php artisan update:coupons
# Waiting for the command to finish
# ...
# Command finished, now we run the next one
~ php artisan send:mail
```

However, you can instruct the OS to run the commands in the background so that you can continue pushing more commands even if the other ones haven't finished yet:

但是，你可以指示操作系统在后台运行命令，以便您可以继续推送更多命令，即使其他命令尚未完成：

```php
~ php artisan update:coupons &
~ php artisan send:mail &
```

Using the ampersand at the end of a command lets you continue pushing commands without having to wait for the initial ones to finish.

使用命令末尾的＆符号可以继续推送命令，而无需等待初始化完成。

The `run()` method checks the value of the `runInBackground` property and decides which method to call next, `runCommandInForeground()` or `runCommandInBackground()`.

`run()` 方法检查 `runInBackground` 属性的值，并决定下一个调用哪个方法`runCommandInForeground()` 还是 `runCommandInBackground()`。

In case the command is to be run in the foreground the rest is simple:

如果命令要在前台运行，其余部分就简单了：

```php
$this->callBeforeCallbacks($container);

(new Process(
    $this->buildCommand(), base_path(), null, null, null
))->run();

$this->callAfterCallbacks($container);
```

Laravel executes any before-callbacks, sends the command to the OS, and finally executes any after-callbacks.

Laravel执行任意before-callbacks，将命令发送到OS，最后执行任意before-callbacks。

However, if the command is to run the background Laravel calls `callBeforeCallbacks()`, sends the command, but doesn't call the after-callbacks, the reason is as you might think, because the command will be executed in the background so if we call `callAfterCallbacks()` at this point it won't be running **after** the command finishes, it'll run once the command is sent to the OS.

但是，如果命令是在后台运行Laravel调用 `callBeforeCallbacks()`，发送命令，但不调用after-callbacks，原因正如你所想的，因为该命令将在后台执行 如果我们在这个时候调用 `callAfterCallbacks()` ，那么在**命令完成之后，它将不会运行**，一旦命令被发送到操作系统，它就会运行。
 
#### So no after-callbacks are executed when we run commands in the background?

#### 那么当我们在后台运行命令时，没有执行回after-callbacks？

They run, laravel does that using another command that runs after the original one finishes:

他们运行了，laravel在原来的一个命令完成后使用另一个命令运行：

```php
(php artisan update:coupons ; php artisan schedule:finish eventMutex) &
```

This command will cause a sequence of two commands to run one after another but in the background, so after your `update:coupons` command finishes a `schedule:finish` command will run given the Mutex of the current event, using this ID Laravel locates the event and runs its after-callbacks.

这个命令会导致一系列的两个命令一个接一个地运行，但在后台运行，所以在你的 `update:coupons` 命令完成一个 `schedule:finish` 命令后，会运行给定当前事件的 Mutex，使用这个ID Laravel 查找事件并运行其回调。

# Building the command string

# 构建命令字符串

When the scheduler calls the `runCommandInForeground()` or `runCommandInBackground()`methods, a `buildCommand()` is called to build the actual command that the OS will run, this method simply does the following:

当调度程序调用 `runCommandInForeground()` 或 `runCommandInBackground()` 方法时，调用一个`buildCommand()` 来构建操作系统将运行的实际命令，这个方法只需执行以下操作：

```php
return (new CommandBuilder)->buildCommand($this);
```

To build the command, the following configurations need to be known:

- The command mutex
- The location that output should be sent to
- Determine if the output should be appended
- The user to run the command under
- Background vs Foreground

构建命令，需要知道以下配置：

- 命令mutex
- 输出应发送到的位置
- 确定输出是否应该追加
- 运行命令的用户
- 后台还是前台

### The command mutex

### 命令互斥

A mutex is a unique ID set for every command, Laravel uses it mainly to prevent command overlapping which we will discuss later, but it also uses it as a unique ID for the command.

互斥是每个命令的唯一ID集合，Laravel主要使用它来防止命令重叠，我们稍后将讨论，但它也将其用作命令的唯一ID。

Laravel defines the mutex of each command inside the `Event::mutexName()` method:

Laravel在 `Event::mutexName()` 方法里面定义每一个命令的互斥:

```php
return 'framework'.DIRECTORY_SEPARATOR.'schedule-'.sha1($this->expression.$this->command);
```

So it's a combination of the CRON expression of the event as well as the command string.

所以它是事件的CRON表达式和命令字符串的组合。

However, for callback events the mutex is created as follows:

但是，对于回调事件是这样创建互斥的：

```php
return 'framework/schedule-'.sha1($this->description);
```

So to ensure having a correct mutex for your callback event you need to set a description for the command:

所以为了确保你的回调事件有一个正确的互斥，你需要为命令设置一个描述：

```php
$schedule->call(function () {
    DB::table('recent_users')->delete();
})->daily()->description('Clear recent users');
```

### Handling output

### 控制输出

By default the output of commands is sent to `/dev/null` which is a special file that discards data written to it, however if you want to send the command output somewhere you can change that using the `sendOutputTo()` method while defining the command:

默认情况下，命令的输出被发送到 `/dev/null` ，这是一个写入丢弃数据的特殊文件，但是如果你想在某个地方发送命令输出，可以使用 `sendOutputTo()` 定义命令：

```php
$schedule->command('mail:send')->sendOutputTo('/home/scheduler.log');
```

But this will cause the output to overwrite whatever is written to the `scheduler.log` file every time, to append the output instead you can use `appendOutputTo()`. Here's how the command would look like:

但这会导致输出覆盖每次写入 `scheduler.log` 文件的任何东西，如果用追加的方式输出可以使用`appendOutputTo()`。 命令如下所示：

```php
// Append output to file
php artisan mail:send >> /home/scheduler.log 2>&1

// Overwrite file
php artisan mail:send > /home/scheduler.log 2>&1
```

2>&1 instructs the OS to redirect error output to the standard output channel, in short words that means errors and output will be logged into your file.

2>&1 指示操作系统将错误输出重定向到标准输出通道，简而言之，这意味着错误和输出将被记录到您的文件中。

### Using the correct user

### 使用正确的用户

When you set a user to run the command:

当你设置一个用户去执行命令的时候:

```php
$schedule->command('mail:send')->user('forge');
```

Laravel will run the command as follows:

Laravel会像下面这样执行命令:

```php
sudo -u forge -- sh -c 'php artisan mail:send >> /home/scheduler.log 2>&1'
```

### Running in the background

### 在后台允许

As we discussed before, the command string will look like this in case it's desired for it to run in the background:

As we discussed before, the command string will look like this in case it's desired for it to run in the background:

正如我们之前讨论过的，命令字符串将如下所示，它在后台运行：

```php
(php artisan update:coupons ; php artisan schedule:finish eventMutex) &
```

But that's just a short form, here's the complete string that'll actually run:

但这只是一个简短的形式，这里是完整的字符串，实际上可以运行：

```php
(php artisan update:coupons >> /home/scheduler.log 2>&1 ; php artisan schedule:finish eventMutex) > /dev/null 2>&1  &
```

Or this if you don't set it to append output & didn't define a custom destination:

或者如果您没有将其设置为追加输出，也没有定义自定义输出位置：

```php
(php artisan update:coupons > /dev/null 2>&1 ; php artisan schedule:finish eventMutex) > /dev/null 2>&1  &
```

### Sending the output via email

### 通过邮件发送输出

You can choose to send the command output to an email address using the `emailOutputTo()`method:

你可以通过调用 `emailOutputTo()` 方法来将命令输出发送到电子邮件

```php
$schedule->command('mail:send')->emailOutputTo(['myemail@mail.com', 'myOtheremail@mail.com']);
```

You can also use `emailWrittenOutputTo()` instead if you want to only receive emails if there's an output, otherwise you'll receive emails even if now output for you to see, it'll be just a notification that the command ran.

如果有一个输出，你只想接收电子邮件你也可以使用 `emailWrittenOutputTo()` ，否则你会收到电子邮件即使现在输出给你你看到了，它也只是命令运行一个通知。

This method will update the output property of the Scheduled Event and point it to a file in the `storage/logs` directory:

这个方法将更新计划事件的输出属性并将其输出到 `storage/logs` 目录中的文件：

```php
if (is_null($this->output) || $this->output == $this->getDefaultOutput()) {
    $this->sendOutputTo(storage_path('logs/schedule-'.sha1($this->mutexName()).'.log'));
}
```

Notice that this will only work if you haven't already set a custom output destination.

注意，只有在尚未设置自定义输出目标位置时，此操作才会起作用。


Next Laravel will register an after-callback that'll try to locate that file, read its content, and send it to the specified recipients.

接下来，Laravel将注册一个回调，尝试找到该文件，读取其内容，并将其发送到指定的收件人。

```php
$text = file_exists($this->output) ? file_get_contents($this->output) : '';

if ($onlyIfOutputExists && empty($text)) {
    return;
}

$mailer->raw($text, function ($m) use ($addresses) {
    $m->to($addresses)->subject($this->getEmailSubject());
});
```