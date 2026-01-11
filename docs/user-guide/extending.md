---
subtitle: Extending
description: Celery扩展和启动步骤完整指南 - 深入讲解如何自定义消息消费者、使用Bootsteps技术扩展Worker和Consumer蓝图、配置核心组件属性和方法，包含丰富的代码示例和详细技术说明，适合需要深度定制Celery分布式任务队列的开发者。
---

# 扩展和启动步骤

## 自定义消息消费者

您可能希望嵌入自定义的 Kombu 消费者来手动处理消息。

为此，存在一个特殊的 `celery.bootstep.ConsumerStep` 启动步骤类，您只需要定义 `get_consumers` 方法，该方法必须返回一个 `kombu.Consumer` 对象列表，在连接建立时启动：

```python
from celery import Celery
from celery import bootsteps
from kombu import Consumer, Exchange, Queue

my_queue = Queue('custom', Exchange('custom'), 'routing_key')

app = Celery(broker='amqp://')


class MyConsumerStep(bootsteps.ConsumerStep):

    def get_consumers(self, channel):
        return [Consumer(channel,
                         queues=[my_queue],
                         callbacks=[self.handle_message],
                         accept=['json'])]

    def handle_message(self, body, message):
        print('Received message: {0!r}'.format(body))
        message.ack()
app.steps['consumer'].add(MyConsumerStep)

def send_me_a_message(who, producer=None):
    with app.producer_or_acquire(producer) as producer:
        producer.publish(
            {'hello': who},
            serializer='json',
            exchange=my_queue.exchange,
            routing_key='routing_key',
            declare=[my_queue],
            retry=True,
        )

        
if __name__ == '__main__':
    send_me_a_message('world!')
```

!!! note

    Kombu 消费者可以使用两种不同的消息回调分发机制。第一种是 `callbacks` 参数，它接受具有 `(body, message)` 签名的回调列表；第二种是 `on_message` 参数，它接受具有 `(message,)` 签名的单个回调。后者不会自动解码和反序列化有效载荷。

    ```python
    def get_consumers(self, channel):
        return [Consumer(channel, queues=[my_queue],
                         on_message=self.on_message)]


    def on_message(self, message):
        payload = message.decode()
        print(
            'Received message: {0!r} {props!r} rawlen={s}'.format(
            payload, props=message.properties, s=len(message.body),
        ))
        message.ack()
    ```

## 蓝图

Bootsteps 是一种为工作器添加功能的技术。一个 bootstep 是一个自定义类，它定义了在工作器的不同阶段执行自定义操作的钩子。每个 bootstep 都属于一个蓝图，工作器目前定义了两个蓝图：**Worker** 和 **Consumer**

<figure markdown="span">
![worker_graph_full](../assets/worker_graph_full.png)
<figcaption>
Worker 和 Consumer 蓝图中的 Bootsteps。从下往上开始，
worker 蓝图中的第一步是 Timer，最后一步是启动 Consumer 蓝图，
然后建立代理连接并开始消费消息。
</figcaption>
</figure>

## Worker

Worker 是第一个启动的蓝图，随之启动的还有主要组件，如事件循环、处理池以及用于 ETA 任务和其他定时事件的计时器。

当 worker 完全启动后，它会继续执行 Consumer 蓝图，该蓝图设置了任务的执行方式，连接到 broker 并启动消息消费者。

`celery.worker.WorkController` 是核心的 worker 实现，包含您可以在 bootstep 中使用的多个方法和属性。

### 属性

<table>
<thead>
<tr>
<th>属性</th>
<th>描述</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>app</code></td>
<td>当前的 app 实例。</td>
</tr>
<tr>
<td><code>hostname</code></td>
<td>worker 的节点名称（例如：<code>worker1@example.com</code>）</td>
</tr>
<tr>
<td><code>blueprint</code></td>
<td>这是 worker 的 <code>celery.bootsteps.Blueprint</code>。</td>
</tr>
<tr>
<td><code>hub</code></td>
<td>
事件循环对象 (<code>kombu.asynchronous.Hub</code>)。您可以使用它在事件循环中注册回调函数。<br/><br/>
这仅受支持异步 I/O 的传输（amqp、redis）支持，在这种情况下，应设置 <code>worker.use_eventloop</code> 属性。<br/><br/>
您的 worker bootstep 必须要求 Hub bootstep 才能使用此功能：

```python
class WorkerStep(bootsteps.StartStopStep):
    requires = {'celery.worker.components:Hub'}
```
</td>
</tr>
<tr>
<td><code>pool</code></td>
<td>
当前进程/eventlet/gevent/线程池。参见 <code>celery.concurrency.base.BasePool</code>。<br/><br/>
您的 worker bootstep 必须要求 Pool bootstep 才能使用此功能：

```python
class WorkerStep(bootsteps.StartStopStep):
    requires = {'celery.worker.components:Pool'}
```
</td>
</tr>
<tr>
<td><code>timer</code></td>
<td>
<code>kombu.asynchronous.timer.Timer</code> 用于调度函数。<br/><br/>
您的 worker bootstep 必须要求 Timer bootstep 才能使用此功能：

```python
class WorkerStep(bootsteps.StartStopStep):
    requires = {'celery.worker.components:Timer'}
```
</td>
</tr>
<tr>
<td><code>statedb</code></td>
<td>
<code>celery.worker.state.Persistent</code> 用于在 worker 重启之间
持久化状态。<br/><br/>
这仅在启用 <code>statedb</code> 参数时定义。<br/><br/>
您的 worker bootstep 必须要求 <code>Statedb</code> bootstep 才能使用此功能：

```python
class WorkerStep(bootsteps.StartStopStep):
    requires = {'celery.worker.components:Statedb'}
```
</td>
</tr>
<tr>
<td><code>autoscaler</code></td>
<td>
<code>celery.worker.autoscaler.Autoscaler</code> 用于自动增长
和缩减池中的进程数量。<br/><br/>
这仅在启用 <code>autoscale</code> 参数时定义。<br/><br/>
您的 worker bootstep 必须要求 <code>Autoscaler</code> bootstep 才能使用此功能：

```python
class WorkerStep(bootsteps.StartStopStep):
    requires = ('celery.worker.components:Autoscaler',)
```
</td>
</tr>
<tr>
<td><code>autoreloader</code></td>
<td>
<code>celery.worker.autoreloader.Autoreloader</code> 用于在文件系统更改时自动重新加载使用代码。<br/><br/>
这仅在启用 <code>autoreload</code> 参数时定义。<br/><br/>
您的 worker bootstep 必须要求 <code>Autoreloader</code> bootstep 才能使用此功能：

```python
class WorkerStep(bootsteps.StartStopStep):
    requires = ('celery.worker.autoreloader:Autoreloader',)
```
</td>
</tr>
</tbody>
</table>

### 示例

一个示例 Worker bootstep 可以是：

```python
from celery import bootsteps


class ExampleWorkerStep(bootsteps.StartStopStep):
    requires = {'celery.worker.components:Pool'}

    def __init__(self, worker, **kwargs):
        print('Called when the WorkController instance is constructed')
        print('Arguments to WorkController: {0!r}'.format(kwargs))

    def create(self, worker):
        # this method can be used to delegate the action methods
        # to another object that implements `start` and `stop`.
        return self

    def start(self, worker):
        print('Called when the worker is started.')

    def stop(self, worker):
        print('Called when the worker shuts down.')

    def terminate(self, worker):
        print('Called when the worker terminates')
```

每个方法都接收当前的 `WorkController` 实例作为第一个参数。

另一个示例可以使用计时器定期唤醒：

```python
from celery import bootsteps


class DeadlockDetection(bootsteps.StartStopStep):
    requires = {'celery.worker.components:Timer'}

    def __init__(self, worker, deadlock_timeout=3600):
        self.timeout = deadlock_timeout
        self.requests = []
        self.tref = None

    def start(self, worker):
        # run every 30 seconds.
        self.tref = worker.timer.call_repeatedly(
            30.0, self.detect, (worker,), priority=10,
        )

    def stop(self, worker):
        if self.tref:
            self.tref.cancel()
            self.tref = None

    def detect(self, worker):
        # update active requests
        for req in worker.active_requests:
            if req.time_start and time() - req.time_start > self.timeout:
                raise SystemExit()
```

### 自定义任务处理日志

Celery worker 会在任务生命周期的各个事件中向 Python 日志子系统发出消息。这些消息可以通过覆盖在 `celery/app/trace.py` 中定义的 `LOG_<TYPE>` 格式字符串来自定义。

例如：

```python
import celery.app.trace

celery.app.trace.LOG_SUCCESS = "This is a custom message"
```

各种格式字符串都提供了任务名称和 ID 用于 `%` 格式化，其中一些还接收额外的字段，如返回值或导致任务失败的异常。这些字段可以在自定义格式字符串中使用，如下所示：

```python
import celery.app.trace

celery.app.trace.LOG_REJECTED = "%(name)r is cursed and I won't run it: %(exc)s"
```

## 消费者

Consumer 蓝图建立与代理（broker）的连接，并且每次连接丢失时都会重新启动。Consumer 启动步骤包括工作节点心跳、远程控制命令消费者，以及重要的任务消费者。

当您创建 consumer 启动步骤时，必须考虑到必须能够重新启动您的蓝图。为 consumer 启动步骤定义了一个额外的 'shutdown' 方法，该方法在工作节点关闭时被调用。

### 属性

<table>
<thead>
<tr>
<th>属性</th>
<th>描述</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>app</code></td>
<td>
当前的应用实例。
</td>
</tr>
<tr>
<td><code>controller</code></td>
<td>
创建此 consumer 的父级 <code>WorkController</code> 对象。
</td>
</tr>
<tr>
<td><code>hostname</code></td>
<td>
工作节点的名称（例如，<code>worker1@example.com</code>）
</td>
</tr>
<tr>
<tr>
<td><code>blueprint</code></td>
<td>
这是工作节点的 <code>celery.bootsteps.Blueprint</code>。
</td>
</tr>
<tr>
<td><code>hub</code></td>
<td>
事件循环对象 (<code>kombu.asynchronous.Hub</code>)。您可以使用它在事件循环中注册回调函数。<br/><br/>
这仅支持启用异步I/O的传输（amqp、redis），在这种情况下，应设置 <code>worker.use_eventloop</code> 属性。<br/><br/>
您的工作节点启动步骤必须要求 <code>Hub</code> 启动步骤才能使用此功能：

```python
class WorkerStep(bootsteps.StartStopStep):
    requires = {'celery.worker.components:Hub'}
```
</td>
</tr>
<tr>
<td><code>connection</code></td>
<td>
当前的代理连接 (<code>kombu.Connection</code>)。<br/><br/>
consumer启动步骤必须要求 <code>Connection</code> 启动步骤才能使用此功能：

```python
class Step(bootsteps.StartStopStep):
    requires = {'celery.worker.consumer.connection:Connection'}
```
</td>
</tr>
<tr>
<td><code>event_dispatcher</code></td>
<td>
一个 <code>events.Dispatcher</code> 对象，可用于发送事件。<br/><br/>
consumer 启动步骤必须要求 <code>Events</code> 启动步骤才能使用此功能。

```python
class Step(bootsteps.StartStopStep):
    requires = {'celery.worker.consumer.events:Events'}
```
</td>
</tr>
<tr>
<td><code>gossip</code></td>
<td>
工作节点之间的广播通信 (<code>celery.worker.consumer.gossip.Gossip</code>)。<br/><br/>
consumer 启动步骤必须要求 <code>Gossip</code> 启动步骤才能使用此功能。

```python
class RatelimitStep(bootsteps.StartStopStep):
    """基于集群中工作节点数量进行任务速率限制"""
    requires = {'celery.worker.consumer.gossip:Gossip'}

    def start(self, c):
        self.c = c
        self.c.gossip.on.node_join.add(self.on_cluster_size_change)
        self.c.gossip.on.node_leave.add(self.on_cluster_size_change)
        self.c.gossip.on.node_lost.add(self.on_node_lost)
        self.tasks = [
            self.app.tasks['proj.tasks.add']
            self.app.tasks['proj.tasks.mul']
        ]
        self.last_size = None

    def on_cluster_size_change(self, worker):
        cluster_size = len(list(self.c.gossip.state.alive_workers()))
        if cluster_size != self.last_size:
            for task in self.tasks:
                task.rate_limit = 1.0 / cluster_size
            self.c.reset_rate_limits()
            self.last_size = cluster_size

    def on_node_lost(self, worker):
        # 可能处理心跳过晚，因此尽快唤醒
        # 以查看工作节点是否恢复
        self.c.timer.call_after(10.0, self.on_cluster_size_change)
```

<b>回调函数</b>

<ul>
<li>
<code>&lt;set&gt; gossip.on.node_join</code><br/>

每当新节点加入集群时调用，提供一个 <code>events.state.Worker</code> 实例。
</li>
<li>
<code>&lt;set&gt; gossip.on.node_leave</code><br/>

每当新节点离开集群（关闭）时调用，
提供一个 <code>celery.events.state.Worker</code> 实例。
</li>
<li>
<code>&lt;set&gt; gossip.on.node_lost</code><br/>
每当集群中的工作节点实例心跳丢失时调用（心跳未及时接收或处理），提供一个 <code>celery.events.state.Worker</code> 实例。<br/>
这并不一定意味着工作节点实际上已离线，因此如果默认的心跳超时不够，请使用超时机制。
</li>
</ul>
</td>
</tr>
<tr>
<td><code>pool</code></td>
<td>
当前的进程/eventlet/gevent/线程池。参见 <code>celery.concurrency.base.BasePool</code>。
</td>
</tr>
<tr>
<td><code>timer</code></td>
<td>
用于调度函数的 <code>celery.utils.timer2.Schedule</code>。
</td>
</tr>
<tr>
<td><code>heart</code></td>
<td>
负责发送工作节点事件心跳(<code>celery.worker.heartbeat.Heart</code>)。<br/><br/>
您的 consumer 启动步骤必须要求 <code>Heart</code> 启动步骤才能使用此功能：

```python
class Step(bootsteps.StartStopStep):
    requires = {'celery.worker.consumer.heart:Heart'}
```
</td>
</tr>
<tr>
<td><code>task_consumer</code></td>
<td>
用于消费任务消息的 <code>kombu.Consumer</code> 对象。<br/><br/>
您的 consumer 启动步骤必须要求 <code>Tasks</code> 启动步骤才能使用此功能：

```python
class Step(bootsteps.StartStopStep):
    requires = {'celery.worker.consumer.tasks:Tasks'}
```
</td>
</tr>
<tr>
<td><code>strategies</code></td>
<td>
每个注册的任务类型在此映射中都有一个条目，其中值用于执行此任务类型的传入消息（任务执行策略）。此映射由Tasks启动步骤在 consumer 启动时生成：

```python
for name, task in app.tasks.items():
    strategies[name] = task.start_strategy(app, consumer)
    task.__trace__ = celery.app.trace.build_tracer(
        name, task, loader, hostname
    )
```

您的 consumer 启动步骤必须要求 <code>Tasks</code> 启动步骤才能使用此功能：

```python
class Step(bootsteps.StartStopStep):
    requires = {'celery.worker.consumer.tasks:Tasks'}
```
</td>
</tr>
<tr>
<td><code>task_buckets</code></td>
<td>
一个 <code>collections.defaultdict</code>，用于按类型查找任务的速率限制。此字典中的条目可以是None（无限制）或实现 <code>consume(tokens)</code> 和 <code>expected_time(tokens)</code> 的 <code>~kombu.utils.limits.TokenBucket</code> 实例。<br/><br/>
TokenBucket实现了[令牌桶算法](https://en.wikipedia.org/wiki/Token_bucket)，但任何算法 只要符合相同的接口并定义上述两个方法都可以使用。
</td>
</tr>
<tr>
<td><code>qos</code></td>
<td>
<code>kombu.common.QoS</code> 对象可用于更改任务通道当前的 prefetch_count 值：

```python
# 在下个周期增加
consumer.qos.increment_eventually(1)
# 在下个周期减少
consumer.qos.decrement_eventually(1)
consumer.qos.set(10)
```
</td>
</tr>
</tbody>
</table>

### 方法

| 方法                                                                                     | 描述                                                              |
|----------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| `reset_rate_limits()`                                                                  | 更新所有注册任务类型的 `task_buckets` 映射。                                  |
| `bucket_for_task(type, Bucket=TokenBucket)`                                            | 使用任务的 `task.rate_limit` 属性为任务创建速率限制桶。                           |
| `add_task_queue(name, exchange=None, exchange_type=None, routing_key=None, **options)` | 添加新的队列以从中消费。这将在连接重启时持久化。                                        |
| `cancel_task_queue(name)`                                                              | 按名称停止从队列消费。这将在连接重启时持久化。                                         |
| `apply_eta_task(request)`                                                              | 根据 `request.eta` 属性调度ETA任务执行。 (`celery.worker.request.Request`) |

## 安装启动步骤

`app.steps['worker']` 和 `app.steps['consumer']` 可以被修改来添加新的启动步骤：

```pycon
>>> app = Celery()
>>> app.steps['worker'].add(MyWorkerStep)  # < 添加类，不要实例化
>>> app.steps['consumer'].add(MyConsumerStep)

>>> app.steps['consumer'].update([StepA, StepB])

>>> app.steps['consumer']
{step:proj.StepB{()}, step:proj.MyConsumerStep{()}, step:proj.StepA{()}
```

这里的步骤顺序并不重要，因为顺序是由产生的依赖图（`Step.requires`）决定的。

为了说明如何安装启动步骤以及它们如何工作，这是一个示例步骤，打印一些无用的调试信息。它可以同时作为 worker 和 consumer 的启动步骤添加：


```python
from celery import Celery
from celery import bootsteps

class InfoStep(bootsteps.Step):

    def __init__(self, parent, **kwargs):
        # 在这里我们可以准备 Worker/Consumer 对象
        # 以任何我们想要的方式，设置属性默认值等等。
        print('{0!r} is in init'.format(parent))

    def start(self, parent):
        # 我们的步骤与所有其他 Worker/Consumer
        # 启动步骤一起启动。
        print('{0!r} is starting'.format(parent))

    def stop(self, parent):
        # Consumer 每次重启时（即连接丢失）以及关闭时都会调用 stop。
        # Worker 仅在关闭时调用 stop。
        print('{0!r} is stopping'.format(parent))

    def shutdown(self, parent):
        # shutdown 由 Consumer 在关闭时调用，
        # Worker 不会调用它。
        print('{0!r} is shutting down'.format(parent))

    app = Celery(broker='amqp://')
    app.steps['worker'].add(InfoStep)
    app.steps['consumer'].add(InfoStep)
```

安装此步骤后启动 worker 将给我们以下日志：

```text
<Worker: w@example.com (initializing)> is in init
<Consumer: w@example.com (initializing)> is in init
[2013-05-29 16:18:20,544: WARNING/MainProcess]
    <Worker: w@example.com (running)> is starting
[2013-05-29 16:18:21,577: WARNING/MainProcess]
    <Consumer: w@example.com (running)> is starting
<Consumer: w@example.com (closing)> is stopping
<Worker: w@example.com (closing)> is stopping
<Consumer: w@example.com (terminating)> is shutting down
```

`print` 语句将在 worker 初始化后重定向到日志子系统，因此 "is starting" 行带有时间戳。您可能会注意到在关闭时不再发生这种情况，这是因为 `stop` 和 `shutdown` 方法在 *信号处理器* 内部调用，在这样的处理器中使用日志记录是不安全的。
Python 日志记录模块不是 [`reentrant`](https://docs.celeryq.dev/en/stable/glossary.html#term-reentrant)：意味着您不能中断函数然后稍后再次调用它。重要的是您编写的 `stop` 和 `shutdown` 方法也是 [`reentrant`](https://docs.celeryq.dev/en/stable/glossary.html#term-reentrant)。

使用 `celery worker --loglevel` 启动 worker 将向我们显示有关启动过程的更多信息：

```text
[2013-05-29 16:18:20,509: DEBUG/MainProcess] | Worker: Preparing bootsteps.
[2013-05-29 16:18:20,511: DEBUG/MainProcess] | Worker: Building graph...
<celery.apps.worker.Worker object at 0x101ad8410> is in init
[2013-05-29 16:18:20,511: DEBUG/MainProcess] | Worker: New boot order:
    {Hub, Pool, Timer, StateDB, Autoscaler, InfoStep, Beat, Consumer}
[2013-05-29 16:18:20,514: DEBUG/MainProcess] | Consumer: Preparing bootsteps.
[2013-05-29 16:18:20,514: DEBUG/MainProcess] | Consumer: Building graph...
<celery.worker.consumer.Consumer object at 0x101c2d8d0> is in init
[2013-05-29 16:18:20,515: DEBUG/MainProcess] | Consumer: New boot order:
    {Connection, Mingle, Events, Gossip, InfoStep, Agent,
     Heart, Control, Tasks, event loop}
[2013-05-29 16:18:20,522: DEBUG/MainProcess] | Worker: Starting Hub
[2013-05-29 16:18:20,522: DEBUG/MainProcess] ^-- substep ok
[2013-05-29 16:18:20,522: DEBUG/MainProcess] | Worker: Starting Pool
[2013-05-29 16:18:20,542: DEBUG/MainProcess] ^-- substep ok
[2013-05-29 16:18:20,543: DEBUG/MainProcess] | Worker: Starting InfoStep
[2013-05-29 16:18:20,544: WARNING/MainProcess]
    <celery.apps.worker.Worker object at 0x101ad8410> is starting
[2013-05-29 16:18:20,544: DEBUG/MainProcess] ^-- substep ok
[2013-05-29 16:18:20,544: DEBUG/MainProcess] | Worker: Starting Consumer
[2013-05-29 16:18:20,544: DEBUG/MainProcess] | Consumer: Starting Connection
[2013-05-29 16:18:20,559: INFO/MainProcess] Connected to amqp://guest@127.0.0.1:5672//
[2013-05-29 16:18:20,560: DEBUG/MainProcess] ^-- substep ok
[2013-05-29 16:18:20,560: DEBUG/MainProcess] | Consumer: Starting Mingle
[2013-05-29 16:18:20,560: INFO/MainProcess] mingle: searching for neighbors
[2013-05-29 16:18:21,570: INFO/MainProcess] mingle: no one here
[2013-05-29 16:18:21,570: DEBUG/MainProcess] ^-- substep ok
[2013-05-29 16:18:21,571: DEBUG/MainProcess] | Consumer: Starting Events
[2013-05-29 16:18:21,572: DEBUG/MainProcess] ^-- substep ok
[2013-05-29 16:18:21,572: DEBUG/MainProcess] | Consumer: Starting Gossip
[2013-05-29 16:18:21,577: DEBUG/MainProcess] ^-- substep ok
[2013-05-29 16:18:21,577: DEBUG/MainProcess] | Consumer: Starting InfoStep
[2013-05-29 16:18:21,577: WARNING/MainProcess]
    <celery.worker.consumer.Consumer object at 0x101c2d8d0> is starting
[2013-05-29 16:18:21,578: DEBUG/MainProcess] ^-- substep ok
[2013-05-29 16:18:21,578: DEBUG/MainProcess] | Consumer: Starting Heart
[2013-05-29 16:18:21,579: DEBUG/MainProcess] ^-- substep ok
[2013-05-29 16:18:21,579: DEBUG/MainProcess] | Consumer: Starting Control
[2013-05-29 16:18:21,583: DEBUG/MainProcess] ^-- substep ok
[2013-05-29 16:18:21,583: DEBUG/MainProcess] | Consumer: Starting Tasks
[2013-05-29 16:18:21,606: DEBUG/MainProcess] basic.qos: prefetch_count->80
[2013-05-29 16:18:21,606: DEBUG/MainProcess] ^-- substep ok
[2013-05-29 16:18:21,606: DEBUG/MainProcess] | Consumer: Starting event loop
[2013-05-29 16:18:21,608: WARNING/MainProcess] celery@example.com ready.
```

## 命令行程序

### 添加新的命令行选项

#### 命令特定选项

您可以通过修改应用程序实例的 `user_options` 属性来向 `worker`、`beat` 和 `events` 命令添加额外的命令行选项。

Celery 命令使用 `click` 模块来解析命令行参数，因此要添加自定义参数，您需要向相关集合添加 `click.Option` 实例。

向 `celery worker` 命令添加自定义选项的示例：

```python
from celery import Celery
from click import Option

app = Celery(broker='amqp://')

app.user_options['worker'].add(Option(('--enable-my-option',),
                                      is_flag=True,
                                      help='Enable custom option.'))
```

所有引导步骤现在都将此参数作为关键字参数传递给 `Bootstep.__init__`：

```python
from celery import bootsteps

class MyBootstep(bootsteps.Step):

    def __init__(self, parent, enable_my_option=False, **options):
        super().__init__(parent, **options)
        if enable_my_option:
            party()

            
app.steps['worker'].add(MyBootstep)
```

#### 预加载选项

`celery` 伞形命令支持'预加载选项'的概念。这些是传递给所有子命令的特殊选项。

您可以添加新的预加载选项，例如指定配置模板：

```python
from celery import Celery
from celery import signals
from click import Option

app = Celery()

app.user_options['preload'].add(Option(('-Z', '--template'),
                                       default='default',
                                       help='Configuration template to use.'))

@signals.user_preload_options.connect
def on_preload_parsed(options, **kwargs):
    use_template(options['template'])
```

##### 添加新的 `celery` 子命令

可以通过使用 [setuptools 入口点](https://setuptools.readthedocs.io/en/latest/setuptools.html#dynamic-discovery-of-services-and-plugins) 向 `celery` 伞形命令添加新命令。

入口点是特殊的元数据，可以添加到包的 `setup.py` 程序中，然后在安装后使用 `importlib` 模块从系统中读取。

Celery 识别 `celery.commands` 入口点来安装额外的子命令，其中入口点的值必须指向有效的 click 命令。

这就是 `Flower` 监控扩展如何通过向 `setup.py` 添加入口点来添加 `celery flower` 命令的方式：

```python
setup(
    name='flower',
    entry_points={
        'celery.commands': [
           'flower = flower.command:flower',
        ],
    }
)
```

命令定义由等号分隔为两部分，第一部分是子命令的名称（flower），第二部分是实现命令的函数的完全限定符号路径：

```text
flower.command:flower
```

模块路径和属性名称应如上所示用冒号分隔。

在 `flower/command.py` 模块中，命令函数可以定义如下：

```python
import click

@click.command()
@click.option('--port', default=8888, type=int, help='Webserver port')
@click.option('--debug', is_flag=True)
def flower(port, debug):
    print('Running our command')
```

## Worker API

`kombu.asynchronous.Hub` - 工作者的异步事件循环

当使用amqp或redis代理传输时，工作者使用异步I/O。最终目标是所有传输都使用事件循环，但这需要一些时间，因此其他传输仍然使用基于线程的解决方案。

| Method                             | Description                                                                                                                                                                                 |
|------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `add(fd, callback, flags)`         | -                                                                                                                                                                                           |
| `add_reader(fd, callback, *args)`  | 添加回调函数，当 `fd` 可读时调用。回调将保持注册状态，直到显式移除使用 `hub.remove(fd)`，或者文件描述符因不再有效而自动丢弃。注意，在任何给定的文件描述符上，一次只能注册一个回调函数，因此第二次调用 `add` 将移除之前为该文件描述符注册的任何回调函数。文件描述符可以是任何支持 `fileno` 方法的类文件对象，也可以是文件描述符编号（int）。 |
| `add_writer(fd, callback, *args)`  | 添加回调函数，当 `fd` 可写时调用。另请参见上面的 `hub.add_reader` 注释。                                                                                                                                            |
| `remove(fd)`                       | 从循环中移除文件描述符 `fd` 的所有回调函数。                                                                                                                                                                   |

### Timer - 调度事件

| Method                                                            | 
|-------------------------------------------------------------------| 
| `call_after(secs, callback, args=(), kwargs=(), priority=0)`      | 
| `call_repeatedly(secs, callback, args=(), kwargs=(), priority=0)` | 
| `call_at(eta, callback, args=(), kwargs=(), priority=0)`          |
