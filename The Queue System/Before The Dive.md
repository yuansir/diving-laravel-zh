Laravel receives a request, does some work, and then returns a response to the user, this is the normal synchronous workflow of a web server handling a request, but sometimes you need to do some work behind the scenes that doesn't interrupt or slowdown this flow, say for example sending an invoice email to the user after making an order, you don't want the user to wait until the mail server receives the request, builds the email message, and then dispatches it to the user, instead you'd want to show a "Thank You!" screen to the user so that he can continue with his life while the email is being prepared and sent in the background.

Laravel接收请求，做一些工作，然后向用户返回响应，这是处理请求的Web服务器的正常同步工作流程，但有时您需要在后台执行不中断或减慢的一些流程，例如在订单之后向用户发送发票电子邮件，你不想让用户等待邮件服务器接收请求,构建电子邮件消息,然后分派给用户，你只要向屏幕发送“谢谢!”给用户,电子邮件在后台准备和发送，他继续做他自己的事。

Laravel is shipped with a built-in queue system that helps you run tasks in the background & configure how the system should react in different situation using a very simple API.

Laravel配有内置的队列系统，可帮助您在后台运行任务，并通过简单的API来配置系统在不同情况下起作用。

You can manage your queue configurations in `config/queue.php`, by default it has a few connections that use different queue drivers, as you can see you can have several queue connections in your project and use multiple queue drivers too.

您可以在 `config/queue.php`中管理队列配置，默认情况下它有使用不同队列驱动的几个连接，您可以看到项目中可以有多个队列连接，也可以使用多个队列驱动程序。

We'll be looking into the different configurations down the road, but let's first take a look at the API:

我们将研究不同的配置，但请先看看API：

```php
Queue::push(new SendInvoice($order));

return redirect('thank-you');
```

The `Queue` facade here proxies to the `queue` container alias, if we take a look at `Queue\QueueServiceProvider` we can see how this alias is registered:

队列`Queue` facade 是 `queue` 容器别名，如果我们看看`Queue\QueueServiceProvider` ，我们可以看到这个别名是如何注册的：

```php
protected function registerManager()
{
    $this->app->singleton('queue', function ($app) {
        return tap(new QueueManager($app), function ($manager) {
            $this->registerConnectors($manager);
        });
    });
}
```

So the `Queue` facade proxies to the `Queue\QueueManager` class that's registered as a singleton in the container, we also register the connectors to different queue drivers that Laravel supports using `registerConnectors()`:

所以 `Queue` facade 代理到在容器中注册为 `Queue\QueueManager` 类的单例，我们还将连接器注册到Laravel所支持使用的`registerConnectors()`的不同队列驱动程序中：

```php
public function registerConnectors($manager)
{
    foreach (['Null', 'Sync', 'Database', 'Redis', 'Beanstalkd', 'Sqs'] as $connector) {
        $this->{"register{$connector}Connector"}($manager);
    }
}
```

This method simply calls the `register{DriverName}Connector` method, for example it registers a Redis connector:

该方法只需调用注册 `register{DriverName}Connector`方法，例如注册一个Redis连接器：

```php
protected function registerRedisConnector($manager)
{
    $manager->addConnector('redis', function () {
        return new RedisConnector($this->app['redis']);
    });
}
```

The `addConnector()` method stores the values to a `QueueManager::$connectors` class property.

A connector is just a class that contains a `connect()` method which creates an instance of the desired driver on demand, here's how the method looks like inside `Queue\Connectors\RedisConnector`:

 `addConnector()` 方法将值存储到 `QueueManager::$connectors` 类属性。
连接器只是一个类，它包含一个 `connect()` 方法，它根据需要创建所需驱动的一个实例，方法看起来像在`Queue\Connectors\RedisConnector`里面:

```php
public function connect(array $config)
{
    return new RedisQueue(
        $this->redis, $config['queue'],
        Arr::get($config, 'connection', $this->connection),
        Arr::get($config, 'retry_after', 60)
    );
}
```

So now The QueueManager is registered into the container and it knows how to connect to the different built-in queue drivers, if we look at that class we'll find a `__call()` magic method at the end:

所以现在QueueManager被注册到容器中，它知道如何连接到不同的内置队列驱动，如果我们看下这个类，我们将在最后找到一个`__call()` 魔术方法：

```php
public function __call($method, $parameters)
{
    return $this->connection()->$method(...$parameters);
}
```

All calls to methods that don't exist in the `QueueManager` class will be sent to the loaded driver, for example when we called the `Queue::push()` method, what happened is that the manager selected the desired queue driver, connected to it, and called the `push` method on that driver.

对 `QueueManager` 类中不存在的方法的所有调用将被发送到加载的驱动中，例如当我们调用 `Queue::push()` 方法时，所发生的是manager选择了连接到它的所需队列驱动 ，并在该驱动上调用 `push` 方法。

#### How does the manager know which driver to pick?

#### 管理器如何知道要选哪个驱动？

Let's take a look at the `connection()` method:

让我们看下 `connection()` 方法

```php
public function connection($name = null)
{
    $name = $name ?: $this->getDefaultDriver();

    if (! isset($this->connections[$name])) {
        $this->connections[$name] = $this->resolve($name);

        $this->connections[$name]->setContainer($this->app);
    }

    return $this->connections[$name];
}
```

When no connection name specified, Laravel will use the default queue connection as per the config files, the `getDefaultDriver()` returns the value of `config/queue.php['default']`:

当没有指定连接名称时，Laravel将根据配置文件使用默认队列连接， `getDefaultDriver()` 返回`config/queue.php['default']`的值：

```php
public function getDefaultDriver()
{
    return $this->app['config']['queue.default'];
}
```

Once a driver name is defined the manager will check if that driver was loaded before, only if it wasn't it starts to connect to that driver and load it using the `resolve()` method:

一旦定义了驱动名称，管理器将检查该驱动是否已被加载，只有当它不是开始连接到该驱动程序并使用 `resolve()` 方法加载它时：

```php
protected function resolve($name)
{
    $config = $this->getConfig($name);

    return $this->getConnector($config['driver'])
                ->connect($config)
                ->setConnectionName($name);
}
```

First it loads the configuration of the selected connection from your `config/queue.php` file, then it locates the connector to the selected driver, calls the `connect()` method on it, and finally sets the connection name for further use.

首先从 `config/queue.php` 文件加载所选连接的配置，然后将连接器定位到所选驱动，调用 `connect()` 方法，最后设置连接名称以供进一步使用。

#### Now we know where to find the push method

#### 现在我们知道在哪里可以找到push方法

Yes, when you do `Queue::push()` you're actually calling the `push` method on the queue driver you're using, each driver handles the different operations in its own way but Laravel provides you a unified interface that you can use to give the queue manager instructions despite of what driver you use.

是的，当您执行 `Queue::push()` 时，您正在使用的队列驱动中调用 `push` 方法，每个驱动以其自己的方式处理不同的操作，但Laravel为您提供了一个统一的接口，您可以使用它告诉队列管理器你使用了什么驱动程序。

#### What if I want to use a driver that's not built-in?

#### 如果我想使用不是内置的驱动程序怎么办？

Easy, you can call `Queue::addConnector()` with the name of your custom driver as well as a closure that explains how a connection to that driver could be acquired, also make sure that your connector implements the `Queue\Connectors\ConnectorInterface` interface.

简单来说，您可以使用自定义驱动的名称调用 `Queue::addConnector()` ，以及一个解释如何获取与该驱动程序的连接的闭包，还要确保您的连接器实现 `Queue\Connectors\ConnectorInterface` 接口。

Once you register the connector, you can use any connection that uses this driver:

注册连接器后，您可以使用任何使用此驱动的连接：

```php
Queue::connection('my-connection')->push(...);
```

Continue to "Preparing Jobs For Queue"

继续“准备队列作业”