---
subtitle: Routing Tasks
description: Celery任务路由指南：学习自动路由、手动路由、优先级设置和广播功能，掌握AMQP基础知识和队列配置技巧，实现高效的任务分发和管理。
---

# 路由任务

## 基础

### 自动路由 {#routing-automatic}

最简单的路由方式是使用 `task_create_missing_queues` 设置（默认启用）。

启用此设置后，未在 `task_queues` 中定义的命名队列将自动创建。这使得执行简单的路由任务变得容易。

假设您有两个服务器 `x` 和 `y` 处理常规任务，还有一个服务器 `z` 仅处理与 feed 相关的任务。您可以使用以下配置::

```python
task_routes = {'feed.tasks.import_feed': {'queue': 'feeds'}}
```

启用此路由后，导入 feed 任务将被路由到 `"feeds"` 队列，而所有其他任务将被路由到默认队列（出于历史原因命名为 `"celery"`）。

或者，您可以使用 glob 模式匹配，甚至正则表达式来匹配 `feed.tasks` 命名空间中的所有任务：

```python
app.conf.task_routes = {'feed.tasks.*': {'queue': 'feeds'}}
```

如果匹配模式的顺序很重要，您应该改用 *items* 格式指定路由器：

```python
task_routes = ([
    ('feed.tasks.*', {'queue': 'feeds'}),
    ('web.tasks.*', {'queue': 'web'}),
    (re.compile(r'(video|image)\.tasks\..*'), {'queue': 'media'}),
],)
```

!!! note

    `task_routes` 设置可以是字典，也可以是路由器对象列表，因此在这种情况下，我们需要将设置指定为包含列表的元组。

安装路由器后，您可以像这样启动服务器 `z` 以仅处理 feeds 队列：

```console
celery -A proj worker -Q feeds
```

您可以指定任意数量的队列，因此您可以让此服务器也处理默认队列：

```console
celery -A proj worker -Q feeds,celery
```

#### 更改默认队列的名称

您可以使用以下配置更改默认队列的名称：

```python
app.conf.task_default_queue = 'default'
```

#### 队列是如何定义的

此功能的目的是为只有基本需求的用户隐藏复杂的 AMQP 协议。然而——您可能仍然对这些队列是如何声明的感兴趣。

名为 `"video"` 的队列将使用以下设置创建：

```javascript
{'exchange': 'video',
 'exchange_type': 'direct',
 'routing_key': 'video'}
```

像 `Redis` 或 `SQS` 这样的非 AMQP 后端不支持交换器，因此它们要求交换器与队列具有相同的名称。使用这种设计确保它也能适用于它们。

### 手动路由

假设您有两个服务器 `x` 和 `y` 处理常规任务，还有一个服务器 `z` 仅处理与 feed 相关的任务，您可以使用以下配置：

```python
from kombu import Queue

app.conf.task_default_queue = 'default'
app.conf.task_queues = (
    Queue('default',    routing_key='task.#'),
    Queue('feed_tasks', routing_key='feed.#'),
)
app.conf.task_default_exchange = 'tasks'
app.conf.task_default_exchange_type = 'topic'
app.conf.task_default_routing_key = 'task.default'
```

`task_queues` 是 `kombu.entity.Queue` 实例的列表。
如果您没有为键设置交换器或交换器类型值，这些值将从 `task_default_exchange` 和 `task_default_exchange_type` 设置中获取。

要将任务路由到 `feed_tasks` 队列，您可以在 `task_routes` 设置中添加一个条目：

```python
task_routes = {
    'feeds.tasks.import_feed': {
        'queue': 'feed_tasks',
        'routing_key': 'feed.import',
    },
}
```

您还可以使用 `apply_async()` 或 `send_task()` 的 `routing_key` 参数覆盖此设置：

```pycon
>>> from feeds.tasks import import_feed
>>> import_feed.apply_async(args=['http://cnn.com/rss'],
...                         queue='feed_tasks',
...                         routing_key='feed.import')
```

要使服务器 `z` 专门从 feed 队列消费，您可以使用 :option:`celery worker -Q` 选项启动它：

```console
celery -A proj worker -Q feed_tasks --hostname=z@%h
```

服务器 `x` 和 `y` 必须配置为从默认队列消费：

```console
celery -A proj worker -Q default --hostname=x@%h
celery -A proj worker -Q default --hostname=y@%h
```

如果您愿意，您甚至可以让您的 feed 处理工作器也处理常规任务，也许在工作量很大的时候：

```console
celery -A proj worker -Q feed_tasks,default --hostname=z@%h
```

如果您有另一个队列但想要添加到另一个交换器，只需指定自定义交换器和交换器类型：

```python
from kombu import Exchange, Queue

app.conf.task_queues = (
    Queue('feed_tasks',    routing_key='feed.#'),
    Queue('regular_tasks', routing_key='task.#'),
    Queue('image_tasks',   exchange=Exchange('mediatasks', type='direct'),
                           routing_key='image.compress'),
)
```

如果您对这些术语感到困惑，您应该阅读 AMQP 的相关资料。

## 特殊路由选项

### RabbitMQ 消息优先级

可以通过设置 `x-max-priority` 参数来配置队列以支持优先级：

```python
from kombu import Exchange, Queue

app.conf.task_queues = [
    Queue('tasks', Exchange('tasks'), routing_key='tasks',
          queue_arguments={'x-max-priority': 10}),
]
```

可以使用 `task_queue_max_priority` 设置来为所有队列设置默认值：

```python
app.conf.task_queue_max_priority = 10
```

也可以使用 `task_default_priority` 设置来为所有任务指定默认优先级：

```python
app.conf.task_default_priority = 5
```

### Redis 消息优先级

虽然 Celery Redis 传输确实尊重优先级字段，但 Redis 本身没有优先级的概念。在尝试使用 Redis 实现优先级之前，请阅读此说明，因为您可能会遇到一些意外行为。

要开始基于优先级调度任务，您需要配置 queue_order_strategy 传输选项。

```python
app.conf.broker_transport_options = {
    'queue_order_strategy': 'priority',
}
```

优先级支持是通过为每个队列创建 n 个列表来实现的。这意味着即使有 10 个（0-9）优先级级别，默认情况下这些级别会被合并为 4 个级别以节省资源。这意味着名为 celery 的队列实际上会被拆分为 4 个队列。

最高优先级的队列将被命名为 celery，而其他队列将有一个分隔符（默认为 `\x06\x16`）并在队列名称后附加其优先级编号。

```python
['celery', 'celery\x06\x163', 'celery\x06\x166', 'celery\x06\x169']
```

如果您想要更多的优先级级别或不同的分隔符，可以设置 priority_steps 和 sep 传输选项：

```python
app.conf.broker_transport_options = {
    'priority_steps': list(range(10)),
    'sep': ':',
    'queue_order_strategy': 'priority',
}
```

上面的配置将为您提供这些队列名称：

```python
['celery', 'celery:1', 'celery:2', 'celery:3', 'celery:4', 'celery:5', 'celery:6', 'celery:7', 'celery:8', 'celery:9']
```

也就是说，请注意这永远不会像在代理服务器级别实现的优先级那样好，最多只能是近似值。但它可能仍然足够满足您的应用程序需求。

## AMQP 基础

### 消息

一条消息由头部和主体组成。Celery 使用头部来存储消息的内容类型和内容编码。内容类型通常是用于序列化消息的序列化格式。主体包含要执行的任务名称、任务ID（UUID）、应用任务的参数以及一些额外的元数据——比如重试次数或预计到达时间（ETA）。

这是一个表示为Python字典的任务消息示例：

```javascript
{'task': 'myapp.tasks.add',
 'id': '54086c5e-6193-4575-8308-dbab76798756',
 'args': [4, 4],
 'kwargs': {}}
```

### 生产者、消费者和代理

发送消息的客户端通常称为*发布者*或*生产者*，而接收消息的实体称为*消费者*。

*代理*是消息服务器，负责将消息从生产者路由到消费者。

在AMQP相关材料中，你很可能会经常看到这些术语。

### 交换机、队列和路由键

1. 消息被发送到交换机。
2. 交换机将消息路由到一个或多个队列。存在几种交换机类型，提供不同的路由方式，或实现不同的消息传递场景。
3. 消息在队列中等待，直到有人消费它。
4. 当消息被确认后，它会被从队列中删除。

发送和接收消息所需的步骤是：

1. 创建一个交换机
2. 创建一个队列
3. 将队列绑定到交换机

Celery 会自动创建 `task_queues` 中队列工作所需的实体（除非队列的 `auto_declare` 设置设为 `False`）。

这是一个包含三个队列的队列配置示例；一个用于视频，一个用于图像，还有一个默认队列用于其他所有内容：

```python
from kombu import Exchange, Queue

app.conf.task_queues = (
    Queue('default', Exchange('default'), routing_key='default'),
    Queue('videos',  Exchange('media'),   routing_key='media.video'),
    Queue('images',  Exchange('media'),   routing_key='media.image'),
)
app.conf.task_default_queue = 'default'
app.conf.task_default_exchange_type = 'direct'
app.conf.task_default_routing_key = 'default'
```

### 交换机类型

交换机类型定义了消息如何通过交换机进行路由。标准中定义的交换机类型有 `direct`、`topic`、`fanout` 和 `headers`。此外，非标准的交换机类型也可作为 RabbitMQ 的插件使用，比如 Michael Bridgen 的 [last-value-cache 插件](https://github.com/squaremo/rabbitmq-lvc-plugin)。

#### 直连交换机

直连交换机通过精确的路由键进行匹配，因此绑定到路由键 `video` 的队列只会接收具有该路由键的消息。

#### 主题交换机

主题交换机使用点分隔的单词和通配符字符来匹配路由键：`*`（匹配单个单词）和 `#`（匹配零个或多个单词）。

对于像 `usa.news`、`usa.weather`、`norway.news` 和 `norway.weather` 这样的路由键，绑定可以是 `*.news`（所有新闻）、`usa.#`（美国的所有项目）或 `usa.weather`（所有美国天气项目）。

#### 相关API命令

**exchange.declare(exchange_name, type, passive, durable, auto_delete, internal)**

通过名称声明一个交换机。

| 关键字         | 描述                                 |
|-------------|------------------------------------|
| passive     | 被动模式意味着不会创建交换机，但你可以使用它来检查交换机是否已存在。 |
| durable     | 持久化交换机是持久的（即，它们在代理重启后仍然存在）。        |
| auto_delete | 这意味着当没有更多队列使用该交换机时，代理将删除它。         |

**queue.declare(queue_name, passive, durable, exclusive, auto_delete)**

通过名称声明一个队列。

独占队列只能由当前连接消费。独占也意味着 `auto_delete`。

**queue.bind(queue_name, exchange_name, routing_key)**

使用路由键将队列绑定到交换机。

未绑定的队列不会接收消息，因此这是必要的。

参见 `amqp.channel.Channel.queue_bind`

**queue.delete(name, if_unused=False, if_empty=False)**

删除一个队列及其绑定。

参见 `amqp.channel.Channel.queue_delete`

**exchange.delete(name, if_unused=False)**

删除一个交换机。

参见 `amqp.channel.Channel.exchange_delete`

!!! note

    声明并不一定意味着"创建"。当你声明时，你*断言*该实体存在且可操作。没有规则规定谁应该最初创建交换机/队列/绑定，无论是消费者还是生产者。通常，第一个需要它的人将是创建它的人。

### 动手实践API

Celery 附带了一个名为 `celery amqp` 的工具，用于通过命令行访问 AMQP API，可以执行管理任务，如创建/删除队列和交换机、清空队列或发送消息。它也可以用于非 AMQP 代理，但不同的实现可能不支持所有命令。

你可以直接在 `celery amqp` 的参数中写入命令，或者不带参数启动它以进入 shell 模式：

```console
celery -A proj amqp
-> connecting to amqp://guest@localhost:5672/.
-> connected.
1>
```

这里的 `1>` 是提示符。数字 1 表示你到目前为止执行的命令数量。输入 `help` 查看可用的命令列表。
它还支持自动补全，因此你可以开始输入命令，然后按 `tab` 键显示可能的匹配列表。

让我们创建一个可以发送消息的队列：

```console
celery -A proj amqp
1> exchange.declare testexchange direct
ok.
2> queue.declare testqueue
ok. queue:testqueue messages:0 consumers:0.
3> queue.bind testqueue testexchange testkey
ok.
```

这创建了直连交换机 `testexchange` 和一个名为 `testqueue` 的队列。该队列使用路由键 `testkey` 绑定到交换机。

从现在开始，所有发送到交换机 `testexchange` 且路由键为 `testkey` 的消息都将被移动到该队列。你可以使用 `basic.publish` 命令发送消息：

```console
4> basic.publish 'This is a message!' testexchange testkey
ok.
```

现在消息已发送，你可以再次检索它。你可以使用 `basic.get` 命令，它以同步方式轮询队列中的新消息（这对于维护任务来说是可以的，但对于服务，你应该使用 `basic.consume`）。

从队列中弹出一条消息：

```console
5> basic.get testqueue
{'body': 'This is a message!',
 'delivery_info': {'delivery_tag': 1,
                   'exchange': u'testexchange',
                   'message_count': 0,
                   'redelivered': False,
                   'routing_key': u'testkey'},
 'properties': {}}
```

AMQP 使用确认来表示消息已成功接收和处理。如果消息未被确认且消费者通道关闭，该消息将被传递给另一个消费者。

注意上面结构中列出的投递标签；在连接通道内，每个接收到的消息都有一个唯一的投递标签，该标签用于确认消息。还要注意投递标签在连接之间不是唯一的，因此在另一个客户端中，投递标签 `1` 可能指向与此通道中不同的消息。

你可以使用 `basic.ack` 确认你收到的消息：

```console
6> basic.ack 1
ok.
```

测试会话结束后，你应该删除创建的实体以进行清理：

```console
7> queue.delete testqueue
ok. 0 messages deleted.
8> exchange.delete testexchange
ok.
```

## 路由任务

### 定义队列

在 Celery 中，可用队列由 `task_queues` 设置定义。

这是一个包含三个队列的示例配置；一个用于视频，一个用于图像，还有一个默认队列用于其他所有任务：

```python
default_exchange = Exchange('default', type='direct')
media_exchange = Exchange('media', type='direct')

app.conf.task_queues = (
    Queue('default', default_exchange, routing_key='default'),
    Queue('videos', media_exchange, routing_key='media.video'),
    Queue('images', media_exchange, routing_key='media.image')
)
app.conf.task_default_queue = 'default'
app.conf.task_default_exchange = 'default'
app.conf.task_default_routing_key = 'default'
```

在这里，`task_default_queue` 将用于路由没有显式路由的任务。

默认的交换机、交换机类型和路由键将用作任务的默认路由值，以及 `task_queues` 中条目的默认值。

还支持对单个队列的多个绑定。这是一个两个路由键都绑定到同一个队列的示例：

```python
from kombu import Exchange, Queue, binding

media_exchange = Exchange('media', type='direct')

CELERY_QUEUES = (
    Queue('media', [
        binding(media_exchange, routing_key='media.video'),
        binding(media_exchange, routing_key='media.image'),
    ]),
)
```

### 指定任务目的地

任务的目的地由以下因素决定（按顺序）：

1. `apply_async()` 的路由参数
2. 在 `Task` 本身上定义的路由相关属性
3. `task_routes` 中定义的 `routers`

最佳实践是不硬编码这些设置，而是通过使用 `routers` 将其作为配置选项；这是最灵活的方法，但仍然可以将合理的默认值设置为任务属性。

### 路由器

路由器是一个决定任务路由选项的函数。

定义新路由器只需要定义一个具有以下签名的函数 `(name, args, kwargs, options, task=None, **kw)`：

```python
def route_task(name, args, kwargs, options, task=None, **kw):
    if name == 'myapp.tasks.compress_video':
        return {'exchange': 'video', 'exchange_type': 'topic', 'routing_key': 'video.compress'}
```

如果您返回 `queue` 键，它将与 `task_queues` 中该队列的已定义设置一起展开：

```javascript
{'queue': 'video', 'routing_key': 'video.compress'}
```

变成 -->

```javascript
{'queue': 'video',
 'exchange': 'video',
 'exchange_type': 'topic',
 'routing_key': 'video.compress'}
```

您可以通过将路由器类添加到 `task_routes` 设置中来安装它们：

```python
task_routes = (route_task,)
```

路由器函数也可以通过名称添加：

```python
task_routes = ('myapp.routers.route_task',)
```

对于像上面路由器示例那样的简单任务名称 -> 路由映射，您可以简单地将字典放入 `task_routes` 以获得相同的行为：

```python
task_routes = {
    'myapp.tasks.compress_video': {
        'queue': 'video',
        'routing_key': 'video.compress',
    },
}
```

然后路由器将按顺序遍历，将在第一个返回真值的路由器处停止，并将其用作任务的最终路由。

您还可以定义多个路由器序列：

```python
task_routes = [
    route_task,
    {
        'myapp.tasks.compress_video': {
            'queue': 'video',
            'routing_key': 'video.compress',
        }
    },
]
```

然后路由器将依次访问，第一个返回值的路由器将被选择。

如果您使用 Redis 或 RabbitMQ，还可以在路由中指定队列的默认优先级。

```python
task_routes = {
    'myapp.tasks.compress_video': {
        'queue': 'video',
        'routing_key': 'video.compress',
        'priority': 10,
    },
}
```

类似地，在任务上调用 `apply_async` 将覆盖该默认优先级。

```python
task.apply_async(priority=0)
```

!!! tip "优先级顺序和集群响应性"

    需要注意的是，由于工作进程预取，如果同时提交一批任务，它们最初可能不按优先级顺序排列。禁用工作进程预取将防止此问题，但可能导致小型快速任务的性能不理想。在大多数情况下，简单地将 `worker_prefetch_multiplier` 减少到 1 是增加系统响应性的更简单、更干净的方法，而无需承担完全禁用预取的成本。

    请注意，使用 redis 代理时，优先级值是反向排序的：0 为最高优先级。

### 广播

Celery 还支持广播路由。
这是一个将任务副本发送给连接到它的所有工作进程的交换机 `broadcast_tasks` 的示例：

```python
from kombu.common import Broadcast

app.conf.task_queues = (Broadcast('broadcast_tasks'),)
app.conf.task_routes = {
    'tasks.reload_cache': {
        'queue': 'broadcast_tasks',
        'exchange': 'broadcast_tasks'
    }
}
```

现在 `tasks.reload_cache` 任务将被发送给从此队列消费的每个工作进程。

这是另一个广播路由的示例，这次使用 `celery beat` 调度：

```python
from kombu.common import Broadcast
from celery.schedules import crontab

app.conf.task_queues = (Broadcast('broadcast_tasks'),)

app.conf.beat_schedule = {
    'test-task': {
        'task': 'tasks.reload_cache',
        'schedule': crontab(minute=0, hour='*/3'),
        'options': {'exchange': 'broadcast_tasks'}
    },
}
```

!!! tip "广播和结果"

    请注意，Celery 结果没有定义如果两个任务具有相同的 task_id 会发生什么。如果同一个任务被分发给多个工作进程，则状态历史可能不会被保留。

    在这种情况下，设置 `task.ignore_result` 属性是一个好主意。