<div align="center" style="padding-bottom: 50px;">
  <br/>
  <img src="https://raw.githubusercontent.com/OptimalBits/bull/master/support/logo%402x.png" width="300" />
  <br/>
</div>

# What is Bull?

Bull 是一个基于 redis 的高效 nodejs 任务队列库

尽管直接使用 redis 中的命令也能实现任务队列的功能，但 bull 将 redis 中低级的 api 进行了封装、整合和强化，使用户在使用任务队列时更加简洁、高效。


如果您之前未接触过任务队列，可能会对任务队列的定位存在疑惑。在许多应用场景中，任务队列能将繁重任务转化为多个微任务或者微服务并协同他们之间的关系使得任务变得更加优雅。而用户只需要关系任务的完成情况而无需关注任务之间的调度情况。


# Getting Started

Bull 是开源的，您可以通过npm 或者 yarn 方便的安装：

```bash
$ npm install bull --save
```

或者

```bash
$ yarn add bull
```
Bull 的工作依赖于 Redis，所以您还需要安装 Redis。在默认情况下，Bull 会访问 Redis 的默认端口和地址`localhost:6379`

# 简单的任务队列

通过Bull 创建一个简单的任务队列实例方式为：

```js
const myFirstQueue = new Bull('my-first-queue');
```

一个任务队列包含三种角色的用户： 任务发布者、任务执行者和任务观察者。




虽然一个任务队列包含三种角色的用户，但是每种角色的用户可以有多个用户。例如在上面的任务队列例子中 `my-first-queue` 中，可以有多个任务发布者，也可以有多个任务执行者，但是所有与当前任务队列相关的用户，都可以被划分为三种用户角色中的一个。需要注意的是，如果任务队列中存在异步的通信机制，即使任务队列中没有任务执行者，任务发布者仍可以向任务队列中添加任务。


同样的，一个任务队列中可以有一个或多个任务执行者，它们将按照一定的顺序从队列中取出任务并执行。其取出任务的先后顺序默认为 `FIFO`，即基于任务优先级。

不同的任务执行者可以公用一个线程，也可以分别使用不同的线程。一旦一台终端或者一个线程链接到 Bull 中指定的 Redis，其就可以被当做是任务执行者去完成任务队列中的任务。

## 任务发布者

任务发布者是一个向任务队列中添加任务的node程序，例如

```js
const myFirstQueue = new Bull('my-first-queue');

const job = await myFirstQueue.add({
  foo: 'bar'
});
```


正如你们所见到的，向队列中添加的任务实际上就是一个json对象，需要注意的是，被添加的任务对象必须是可序列化的，即必须能通过 stringify 方法转化为字符串，否则reids 无法存储该对象。


## 任务执行者


任务执行者实际上是一个可以执行任务队列中任务的一段 node 代码。例如：

```js
const myFirstQueue = new Bull('my-first-queue');

myFirstQueue.process(async (job) => {
  return doSomething(job.data);
});
```

只要任务队列不为空且存在至少一个任务执行者， `process` 就会被执行。由于任务在被分配给任务执行者之后，任务执行者可以离线执行任务。一种可能的情况是，任务在被分配后很久都没有返回任务执行完成的消息，此时 `process` 方法就会一直保持繁忙状态并一直等待所有任务都被执行完成。

在上面的例子中，我们将方法定义为异步方法 `async`，我们强烈建议所有的任务形式都用该形式去定义。如果您的node环境不支持异步方法或者不方便使用 `async / await`，您需要在函数的之后返回 `promise` 对象

任务执行者返回的对象将被存储在被执行的任务对象中以便其他角色访问，例如监听者可以访问该属性监听任务状态。


在某些情况下，您可能希望任务处理的进程被显示的处理，例如显示在进度条上。使用 `progress` 方法可以方便的设定一个任务完成的进度情况：

```js
myFirstQueue.process( async (job) => {
  let progress = 0;
  for(i = 0; i < 100; i++){
    await doSomething(job.data);
    progress += 10;
    job.progress(progress);
  }
});
```

## 任务观察者


最后，您可能希望能监听到任务在执行过程中的状态，例如执行进度，是否被完成等信息，此时需要任务观察者协助。任务观察者分为局部观察者和全局观察者。局部观察者顾名思义，只能观察到指定任务队列中出现的消息。而全局观察者则可以观察到所有消息队列中的消息。您可以使用任务观察者去监听一个任务执行者或者任务发布者。需要注意的是，如果一个任务队列既没有任务执行者也没有任务发布者，您无法抛出一个局部消息，仅能抛出一个全局消息。

```js
const myFirstQueue = new Bull('my-first-queue');

// Define a local completed event
myFirstQueue.on('completed', (job, result) => {
  console.log(`Job completed with result ${result}`);
})
```

## 被执行任务的声明周期


为了能更好的了解并使用 Bull 的功能，您可能需要了解Bull 中被执行任务的声明周期。一旦任务发布者通过 `add` 方法将一个任务添加到任务队列中，该任务的生命周期就开始生效直到任务被执行完成或者任务执行失败。其声明周期如图所示：


![Bull 中任务声明周期](./job-lifecycle_20200924113541.png)



一旦一个任务被添加到任务队列中，其立马会变成“等待”或者“延迟”两个状态中的一个。其中**等待**状态代表一个正常任务等待被任务执行者执行的状态。而**延迟**状态代表一个任务由于某种原因执行超时，其会被放置到待执行任务队列的顶端（**延迟**的任务不会立马被执行，知道任务队列中有可用的任务执行者，且被延迟的任务优先级较高）

下一个状态为**激活**状态。**激活**状态代表一个任务正在被任务执行者执行，一个任务可以被执行者执行无限长时间直到执行者抛出异常或者返回任务执行完成消息。然后根据执行者返回的执行消息，将任务的状态转为**失败**或者**完成**状态。


## 停滞任务

在Bull 中，停滞任务是任务的一种特殊状态，其代表当一个任务执行者执行了很长时间以至于 Bull 任务任务已经被执行完成但是并未收到任务执行完成消息。这种情况在CPU负载非常严重时可能发生。一旦一个任务被识别为停滞任务，基于任务设置中的配置信息，可以选择性的将当前任务移交给另一个空闲的任务执行者或将其标记为**失败**的任务，并进入声明周期中的**延迟** 任务


在正常情况下，只要 node 中负责时间执行的循环负载不是特别繁重或者采用合适的分布式架构，一般不会出现任务停滞 的情况。


# 事件

Bull 中事件队列可以产生一系列时间共用户去处理其各种情况。事件可以是只针对一个消息队列的消息也可以是是针对一组队列的消息。例如只针对一个任务队列的任务执行完成消息：

```js
queue.on('completed', job => {
  console.log(`Job with id ${job.id} has been completed`);
})
```


而针对所有任务队列的全局消息为：


```js
queue.on('global:completed', jobId => {
  console.log(`Job with id ${jobId} has been completed`);
})
```

需要注意的是，局部消息和全局消息在传递参数时并不相同，局部消息会将整个队列消息传递。需要注意的是，全局消息和局部消息在回调时的参数是不同的。对于局部消息来说，其参数为整个消息内容，而对于全局消息来说，其参数只有每个消息对应任务的id。

# 任务队列设置


在初始化一个任务队列时，可以对其进行一些自定义的设置。例如可以自定义 Redis 的账号和密码。全部的设置信息可以查看[这里](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queue) ，此处我们仅介绍几种最为常用的设置，其他的不再详细展开。

## 队列任务流量

通过设置参数 `rateLimited` 可以控制在任意时间段内被处理的任务个数。例如：
```js
// 限制每5秒处理1000个任务
const myRateLimitedQueue = new Queue('rateLimited', {
  limiter: {
    max: 1000,
    duration: 5000
  }
});
```

When a queue hits the rate limit, requested jobs will join the `delayed` queue.
当一个任务被添加到任务队列中，但其流量已经超过阈值时，其声明周期会变更为`延迟`

## 定义任务名称

在天啊及任务时，可以定义被添加任务的名称。定义任务名称不会影响任务处理的逻辑，但是可以让代码的可读性更强。例如：

```js
// Jobs producer
const myJob = await transcoderQueue.add('image', { input: 'myimagefile' });
const myJob = await transcoderQueue.add('audio', { input: 'myaudiofile' });
const myJob = await transcoderQueue.add('video', { input: 'myvideofile' });
```

```js
// Worker
transcoderQueue.process('image', processImage);
transcoderQueue.process('audio', processAudio);
transcoderQueue.process('video', processVideo);
```

需要注意的是，您在向任务队列中添加包含名称的任务时，必须预先定义与当前名称相对应的任务处理者的名称，否则程序会出现异常。

## 沙盒处理

在前面我们介绍到了任务的执行者，在定义任务处理方法时，可以定义任务处理是否为并行处理。如果任务为并行处理，则允许多个处理者同时处理多个任务。虽然任务被并行处理，但是其宿主进程依旧是同一个node进程，所以既是是IO密集型任务，其也可以正常工作。



在实际情况中，更多的情况是CPU占用过多，此时可能导致 node 主进程占用时间过长。此时可以设置同时运行的任务数，避免主进程被卡死。

我们称这种随时可更换的进程为 `沙盒进程`， 当沙盒进程出现异常或者超时时，其不会影响其他进程并可以被主进程随机替换为正常进程。


# 任务类型

在默认情况下，所有被添加到任务队列中的任务类型都是 `FIFO`，即先进后出规则。

## LIFO（先进后出）

Lifo (last in first out) 表示每个被添加的任务都被添加到任务队列的顶部，当有处理者空闲时被添加的任务会被立即执行。

```js
const myJob = await myqueue.add({ foo: 'bar' }, { lifo: true });
```

## Delayed（延迟处理）

同样的，我们可以定义一个任务为延时处理，即至少经过多久再处理该任务。需要注意的是，定义的时间参数为**至少**经过多久再处理。一旦等待时间超过预定义的延时时间并且有空闲的处理者时，当前任务就会被添加到队列的头部并被处理。

```js
// Delayed 5 seconds
const myJob = await myqueue.add({ foo: 'bar' }, { delay: 5000 });
```

## Prioritized （预定义优先级）

当任务被添加到任务队列中时，可以预定义其被处理的优先级，有用较高优先级的任务将被优先处理。最高优先级为1，数字越大，优先级越低。需要注意的是，使用预定义优先级的任务会导致任务排序进而略微影响性能（假设队列中有n个等待的任务，插入新的预定义优先级的任务的时间复杂度为O(n)）

```js
const myJob = await myqueue.add({ foo: 'bar' }, { priority: 3 });
```

## Repeatable （循环任务）

在添加任务时，如果指定属性`repeat` 则表示当前任务为循环任务，其将被反复执行，直到结束条件成立。例如：

```js
// 每过10 秒循环一次，总共循环100次
const myJob = await myqueue.add(
  { foo: 'bar' },
  {
    repeat: {
      every: 10000,
      limit: 100
    }
  }
);

// 在每天的早上3:15 执行一次
paymentsQueue.add(paymentsData, { repeat: { cron: '15 3 * * *' } });
```

在使用循环任务时，您可能需要注意以下几点：

- Bull 中自带循环任务的去重步骤，即当两个循环条件相互重叠时，重叠部分不会重复运行。如果您确实需要不同循环任务之间彼此独立且不需要去重，可以考虑根据任务id设置定时任务。具体请参照：https://github.com/OptimalBits/bull/pull/603
- 如果循环任务被唤醒时没有可用的处理者，则未执行的任务会被积累到下次任务中。
- 如果想要移除一个循环任务，可以使用[removeRepeatable](https://github.com/OptimalBits/bull/blob/master/REFERENCE.md#queueremoverepeatable) 方法移除.

