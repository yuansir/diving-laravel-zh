Every job we push to queue is stored in some storage space sorted by the order of execution, this storage place could be a MySQL database, Redis store, or a 3rd party service like Amazon SQS.

我们推送到队列的每个作业都存储在按执行顺序排序的某些存储空间中，该存储位置可以是MySQL数据库，Redis存储或像Amazon SQS这样的第三方服务。

Later in this dive we're going to explore how workers fetch these jobs and start executing them, but before that let's see how we store our jobs, here are the attributes we keep for every job we store:

在这次深入探究之后，我们将探索worker如何获取这些作业并开始执行这些作业，但在此之前，让我们看看我们如何存储我们的作业，这里是我们为每个我们存储的作业所保留的属性：

- What queue it should be running through
- The number of times this job was attempted (Initially zero)
- The time when the job was reserved by a worker
- The time when the job becomes available for a worker to pickup
- The time when the job was created
- The payload of the job

- 应该运行什么队列
- 此作业尝试次数（最初为零）
- 作业由worker保留的时间
- 作业时间可供worker领取
- 创建作业的时间
- 作业的有效载荷

#### What do you mean by queue?

## 你说的排队是什么意思？

Your application sends several types of jobs to queue, by default all the jobs are put in a single queue; however, you might want to put mail jobs in a different queue and instruct a dedicated worker to run jobs only from this queue, this will ensure that other jobs won't delay sending emails since it has a dedicated worker of its own.

您的应用程序发送很多类型的作业到队列，默认情况下，所有作业都放在单个队列中; 但是，您可能希望将邮件作业放在不同的队列中，并指示专用worker仅从此队列运行作业，这将确保其他作业不会延迟发送电子邮件，因为它具有自己的专用worker。

### The number of attempts

#### 尝试次数

By default the queue manager will keep trying to run a specific job if it fails running, it'll keep doing that forever unless you set a maximum number of attempts, when the job attempts reach that number the job will be marked as failed and workers won't try to run it again. This number starts as zero but we keep incrementing it every time we run the job.

默认情况下，如果运行失败，队列管理器将继续尝试运行特定的作业，除非您设置了最大次数，否则将继续执行此操作，当作业尝试达到该数量时，该作业将被标记为失败，并且worker 不会再尝试运行它。 这个数字从0开始，在每次运行作业时不断增加。

### The Reservation time

#### 预约时间

Once a worker picks a job we mark it as reserved and store the timestamp of when that happened, next time a worker tries to pick a new job it won't pick a reserved one unless the reservation lock is expired, but more on that later.

一旦worker选择了一个作业，我们将其标记为保留，并存储发生的时间戳，下一次worker尝试选择新的作业时，除非保留锁定已过期，否则不会选择保留位。

### The availability time

#### 可用时间

By default a job is available once it's pushed to queue and workers can pick it right away, but sometimes you might need to delay running a job for sometime, you can do that by providing a delay while pushing the job to queue using the `later()` method instead of `push()`:

默认情况下，一旦作业被推送到队列，worker可以立即获取它，但有时您可能需要延迟一段时间的运行作业，您可以通过 `later()` 方法而不是 `push()`来延迟来推送作业：

```php
Queue::later(60, new SendInvoice())
```

The `later()` method uses the `availableAt()` method to determine the availability time:

后者 `later()` 方法使用 `availableAt()` 方法来确定可用性时间：

```php
protected function availableAt($delay = 0)
{
    return $delay instanceof DateTimeInterface
                        ? $delay->getTimestamp()
                        : Carbon::now()->addSeconds($delay)->getTimestamp();
}
```

As you can see you can pass an instance of `DateTimeInterface` to set the exact time, or you can pass the number of seconds and Laravel will calculate the availability time for you under the hood.

您可以看到，您可以传递一个 `DateTimeInterface` 的实例来设置确切的时间，或者您可以传递秒数，Laravel会计算出您的可用时间。

## The payload

#### 有效载荷

The `push()` & `later()` methods use the `createPayload()` method internally to create the information needed to execute the job when it's picked by a worker. You can pass a job to queue in two formats:

 `push()` & `later()` 方法在内部使用 `createPayload()` 方法来创建执行作业所需的信息，当worker选择它时。 您可以通过两种格式将作业传递给队列：

```php
// Pass an object
Queue::push(new SendInvoice($order));

// Pass a string
Queue::push('App\Jobs\SendInvoice@handle', ['order_id' => $orderId])
```

In the second example while the worker is picking the job, Laravel will use the container to create an instance of the given class, this gives you the chance to require any dependencies in the job's constructor.

在第二个例子中，当worker正在选择作业时，Laravel将使用该容器来创建给定类的实例，这样可以让您有机会在作业的构造函数中包含任何依赖。

### Creating the payload of a string job

#### 创建字符串作业的有效负荷

`createPayload()` calls `createPayloadArray()` internally which calls the `createStringPayload()` method in case the job type is non-object:

`createPayload()`在内部调用 `createPayloadArray()`，调用 `createStringPayload()` 方法，以防作业类型为空对象：

```php
protected function createStringPayload($job, $data)
{
    return [
        'displayName' => is_string($job) ? explode('@', $job)[0] : null,
        'job' => $job, 'maxTries' => null,
        'timeout' => null, 'data' => $data,
    ];
}
```

The `displayName` of a job is a string you can use to identify the job that's running, in case of non-object job definitions we use the the job class name as the `displayName`.

Notice also that we store the given data in the job payload.

作业的`displayName` 是可用于标识正在运行的作业的字符串，在空对象作业定义的情况下，我们使用作业类名称作为 `displayName`.

Notice also that we store the given data in the job payload.

还要注意，我们将给定的数据存储在作业有效载荷中。

### Creating the payload of an object job

#### 创建对象作业的有效负载

Here's how an object-based job payload is created:

以下是创建基于对象的作业有效负载的方式：

```php
protected function createObjectPayload($job)
{
    return [
        'displayName' => $this->getDisplayName($job),
        'job' => 'Illuminate\Queue\CallQueuedHandler@call',
        'maxTries' => isset($job->tries) ? $job->tries : null,
        'timeout' => isset($job->timeout) ? $job->timeout : null,
        'data' => [
            'commandName' => get_class($job),
            'command' => serialize(clone $job),
        ],
    ];
}
```

Since we already have the instance of the job we can extract some useful information from it, for example the `getDisplayName()` method looks for a `displayName()` method inside the job instance and if found it uses the return value as the job name, that means you can add such method in your job class to be in control of the name of your job in queue.

由于我们已经有了这个作业的实例，我们可以从中提取一些有用的信息，例如， `getDisplayName()` 方法在作业实例中查找一个 `displayName()` 方法，如果发现它使用返回值作为作业名称，那么 意味着您可以在作业类中添加这样的方法来控制队列中作业的名称。

```php
protected function getDisplayName($job)
{
    return method_exists($job, 'displayName')
                    ? $job->displayName() : get_class($job);
}
```

We can also extract the value of the maximum number a job should be retried and the timeout for the job, if you pass these values as class properties Laravel stores this data into the payload for use by the workers later.

As for the `data` attribute, Laravel stores the class name of the job as well as a serialized version of that job.

如果将这些值作为类属性传递，我们还可以提取作业应该重试的最大数量和作业的超时值，Laravel将此数据存储到有用的worker中以供以后使用。

对于数据属性，Laravel存储作业的类名称以及该作业的序列化版本。

#### Then how can I pass my own data

#### 那么如何传递我自己的数据

In case you chose to pass an object-based job you can't provide a data array, you can store any data you need inside the job instance and it'll be available once un-serialized.

如果您选择传递基于对象的作业，则无法提供数据数组，您可以在作业实例中存储所需的任何数据，并且一旦未序列化，它都是可用的。

#### Why do we pass a different class as the "job" parameter

#### 为什么我们通过不同的类作为“作业”的参数

`Queue\CallQueuedHandler@call` is a special class Laravel uses while running object-based jobs, we'll look into it in a later stage.

`Queue\CallQueuedHandler@call` 是Laravel在运行基于对象的作业时使用的一个特殊类，我们将在稍后的阶段进行研究。

#### Serializing jobs

#### 序列化作业

Serializing a PHP object generates a string that holds information about the class the object is an instance of as well as the state of that object, this string can be used later to re-create the instance.

In our case we serialize the job object in order to be able to easily store it somewhere until a worker is ready to pick it up & run it, while creating the payload for the job we serialize a clone of the job object:

序列化PHP对象会生成一个字符串，该字符串保存有关该对象是该实例的类的信息以及该对象的状态，此字符串以后可用于重新创建该实例。

一般情况下，我们序列化作业对象，以便能够轻松地将其存储在某个位置，直到worker准备好接收并运行它，为作业创建有效负载的同时我们将序列化作业对象的克隆：

```php
serialize(clone $job);
```

#### But why a clone? why not serialize the object itself?

#### 但为什么要克隆？ 为什么不序列化对象本身？

While serializing the job we might need to do some transformation to some of the job properties or properties of any of the instances our job might be using, if we pass the job instance itself transformations will be applied to the original instances while this might not be desired, let's take a look at an example:

在序列化作业时，我们可能需要对我们的作业可能使用的任何实例的某些作业属性或属性进行一些转换，如果我们传递作业实例本身，转换将应用于原始实例，而这可能不是 期望的，让我们来看一个例子：

```php
class SendInvoice
{
    public function __construct(Invoice $Invoice)
    {
        $this->Invoice = $Invoice;
    }
}

class Invoice
{
    public function __construct($pdf, $customer)
    {
        $this->pdf = $pdf;
        $this->customer = $customer;
    }
}
```

While creating the payload for the `SendInvoice` job we're going to serialize that instance and all its child objects, the Invoice object, but PHP doesn't always work well with serializing files and the Invoice object has a property called `$pdf` which holds a file, for that we might want to store that file somewhere, keep a reference to it in our instance while serializing and then when we un-serialize we bring back the file using that reference.

在为 `SendInvoice` 作业创建载荷时，我们将序列化该实例及其所有子对象，发票, 但PHP并不总是能很好地处理序列化文件，并且发票对象具有称为 `$pdf` 的属性，该属性包含 文件，因为我们可能想要将该文件存储在某个地方，在序列化期间保留对我们实例的引用，然后当我们进行非序列化时，我们使用该引用返回文件。

```php
class Invoice
{
    public function __construct($pdf, $customer)
    {
        $this->pdf = $pdf;
        $this->customer = $customer;
    }

    public function __sleep()
    {
        $this->pdf = stream_get_meta_data($this->pdf)['uri'];

        return ['customer', 'pdf'];
    }

    public function __wakeup()
    {
        $this->pdf = fopen($this->pdf, 'a');
    }
}
```

The `__sleep()` method is automatically called before PHP starts serializing our object, in our example we transform our pdf property to hold the path to the PDF resource instead of the resource itself, and inside `__wakup()` we convert that path back to the original value, that way we can safely serialize the object.

Now if we dispatch our job to queue we know that it's going to be serialized under the hood:

在PHP开始序列化我们的对象之前，`__sleep()` 方法被自动调用，在我们的示例中，我们将我们的pdf属性转换为保存到PDF资源的路径而不是资源本身，而在`__wakup()` 中，我们将该路径转换回原始值，这样我们可以安全地序列化对象。

现在如果我们分发作业到队列，将被序列化：

```php
$Invoice = new Invoice($pdf, 'Customer #123');

Queue::push(new SendInvoice($Invoice));

dd($Invoice->pdf);
```

However, if we try to look at the Invoice pdf property after sending the job to queue we'll find that it holds the path to the resource, it doesn't hold the resource anymore, that's because while serializing the Job PHP serialized the original Invoice instance as well since it was passed by reference to the SendInvoice instance.

然而，如果我们在发送作业到队列后尝试查看发票pdf属性，我们会发现它拥有资源的路径，它不再占用资源，这是因为序列化作业时PHP已经序列化了原始发票实例，因为它通过引用传递给SendInvoice实例。

Here's when cloning becomes handy, while cloning the `SendInvoice` object PHP will create a new instance of that object but that new instance will still hold reference to the original Invoice instance, but we can change that:

以下是克隆handy时，克隆 `SendInvoice` 对象时PHP将创建该对象的新实例，但新的实例仍将保留对原始Invoice实例的引用，但是我们可以更改：

```php
class SendInvoice
{
    public function __construct(Invoice $Invoice)
    {
        $this->Invoice = $Invoice;
    }

    public function __clone()
    {
        $this->Invoice = clone $this->Invoice;
    }
}
```

Here we instruct PHP that whenever it clones an instance of the `SendInvoice` object it should use a clone of the invoice property in the new instance not the original one, that way the original Invoice object will not be affected while we serialize.

这里我们介绍PHP只要克隆 `SendInvoice` 对象的一个实例，它应该使用新实例中的Invoice属性的克隆，而不是原始实例，那么原始Invoice对象在序列化时不会受到影响。