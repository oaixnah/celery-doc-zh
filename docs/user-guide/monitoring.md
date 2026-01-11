---
subtitle: Monitoring and Management
description: Celery监控与管理指南 - 详细介绍Celery集群的监控工具和方法，包括Flower实时Web监控器、命令行工具(inspect/control)、事件系统、RabbitMQ和Redis监控，以及实时事件处理和快照功能。
---

# 监控与管理

## 简介

有多种工具可用于监控和检查 Celery 集群。

本文档介绍其中一些工具，以及与监控相关的功能，如事件和广播命令。

## Workers

### 管理命令行工具 (`inspect`/`control`)

`celery` 也可以用来检查和管理工作节点（以及在一定程度上管理任务）。

要列出所有可用的命令，请执行：

```console
celery --help
```

或者要获取特定命令的帮助，请执行：

```console
celery <command> --help
```

### 命令

#### shell

进入 Python shell。

本地变量将包含 `celery` 变量：这是当前的应用。所有已知的任务也会自动添加到本地变量中（除非设置了 `celery shell --without-tasks` 标志）。

如果已安装，将按以下顺序使用 `Ipython`、`bpython` 或常规的 `python`。
您可以使用以下选项强制使用特定的实现：`celery shell --ipython`、`celery shell --bpython` 或 `celery shell --python`。

#### status

列出此集群中的活动节点

```console
celery -A proj status
```

#### result

显示任务的结果

```console
celery -A proj result -t tasks.add 4e196aa4-0141-4601-8138-7aa33db0f577
```

请注意，只要任务不使用自定义结果后端，您就可以省略任务名称。

#### purge

清除所有已配置任务队列中的消息。

此命令将从 `CELERY_QUEUES` 设置中配置的队列中删除所有消息：

!!! warning "此操作无法撤销，消息将被永久删除！"

```console
celery -A proj purge
```

您还可以使用 `-Q` 选项指定要清除的队列：

```console
celery -A proj purge -Q celery,foo,bar
```

并使用 `-X` 选项排除队列不被清除：

```console
celery -A proj purge -X celery
```

#### inspect active

列出活动任务

```console
celery -A proj inspect active
```

这些是当前正在执行的所有任务。

#### inspect scheduled 

列出已调度的 ETA 任务

```console
celery -A proj inspect scheduled
```

这些是当任务设置了 `eta` 或 `countdown` 参数时由工作进程保留的任务。

#### inspect reserved

列出保留任务

```console
celery -A proj inspect reserved
```

这将列出工作进程已预取的所有任务，这些任务当前正在等待执行（不包括设置了 ETA 值的任务）。

#### inspect revoked

列出已撤销任务的历史记录

```console
celery -A proj inspect revoked
```

这将列出所有已被撤销的任务。

#### inspect registered

列出已注册任务

```console
celery -A proj inspect registered
```

这将列出系统中注册的所有任务。

#### inspect stats

显示工作进程统计信息（参见 [工作进程指南 - 统计信息](workers.md#worker-statistics){target="_blank"}）

```console
celery -A proj inspect stats
```

#### inspect query_task

按 ID 显示任务信息。

任何在此 ID 集合中有保留/活动任务的工作进程都将响应状态和信息。

```console
celery -A proj inspect query_task e9f6c8f0-fec9-4ae8-a8c6-cf8c8451d4f8
```

您还可以查询多个任务的信息：

```console
celery -A proj inspect query_task id1 id2 ... idN
```

#### control enable_events

启用事件

```console
celery -A proj control enable_events
```

#### control disable_events

禁用事件

```console
celery -A proj control disable_events
```

#### migrate

将任务从一个代理迁移到另一个代理（**实验性**）。

```console
celery -A proj migrate redis://localhost amqp://localhost
```

此命令将把一个代理上的所有任务迁移到另一个代理。由于此命令是新的且具有实验性，您应该确保在继续之前备份数据。

!!! note

    所有 `inspect` 和 `control` 命令都支持 `celery inspect --timeout` 参数，这是等待响应的秒数。如果由于延迟而无法获得响应，您可能需要增加此超时时间。

#### 指定目标节点

默认情况下，inspect 和 control 命令对所有工作进程进行操作。您可以使用 `celery inspect --destination` 参数指定单个或一组工作进程：

```console
celery -A proj inspect -d w1@e.com,w2@e.com reserved

celery -A proj control -d w1@e.com,w2@e.com enable_events
```

### Flower: 实时 Celery Web 监控器

Flower 是一个基于 Web 的实时监控和管理工具，专为 Celery 设计。它正在积极开发中，但已经是一个必不可少的工具。作为 Celery 的推荐监控器，它取代了 Django-Admin 监控器、`celerymon` 和基于 `ncurses` 的监控器。

Flower 的发音类似于 "flow"，但如果您喜欢，也可以使用植物版本的发音。

#### 特性

- 使用 Celery 事件进行实时监控

    - 任务进度和历史记录
    - 显示任务详细信息的能力（参数、开始时间、运行时间等）
    - 图表和统计信息

- 远程控制

    - 查看工作器状态和统计信息
    - 关闭和重启工作器实例
    - 控制工作器池大小和自动缩放设置
    - 查看和修改工作器实例消费的队列
    - 查看当前运行的任务
    - 查看计划任务（ETA/倒计时）
    - 查看保留和撤销的任务
    - 应用时间和速率限制
    - 配置查看器
    - 撤销或终止任务

- HTTP API

    - 列出工作器
    - 关闭工作器
    - 重启工作器池
    - 扩展工作器池
    - 缩小工作器池
    - 自动缩放工作器池
    - 开始从队列消费
    - 停止从队列消费
    - 列出任务
    - 列出（已见到的）任务类型
    - 获取任务信息
    - 执行任务
    - 按名称执行任务
    - 获取任务结果
    - 更改任务的软硬时间限制
    - 更改任务的速率限制
    - 撤销任务

- OpenID 认证

**截图**

![Dashboard](../assets/dashboard.png)

更多 [截图](https://github.com/mher/flower/tree/master/docs/screenshots)：

#### 使用方法

您可以使用 pip 安装 Flower：

```console
pip install flower
```    

运行 flower 命令将启动一个您可以访问的 Web 服务器：

```console
celery -A proj flower
```

默认端口是 http://localhost:5555，但您可以使用 [`--port`](https://flower.readthedocs.io/en/latest/config.html#port) 参数更改此设置：

```console
celery -A proj flower --port=5555
```

代理 URL 也可以通过 `celery --broker` 参数传递：

```console
celery --broker=amqp://guest:guest@localhost:5672// flower
# or
celery --broker=redis://guest:guest@localhost:6379/0 flower
```

然后，您可以在 Web 浏览器中访问 flower：

```console
open http://localhost:5555
```

Flower 具有比此处详述的更多功能，包括授权选项。请查看 [官方文档](https://flower.readthedocs.io/en/latest/) 以获取更多信息。

### celery events: Curses 监视器

`celery events` 是一个简单的 curses 监视器，显示任务和工作者历史记录。您可以检查任务的结果和回溯信息，它还支持一些管理命令，如速率限制和关闭工作者。这个监视器最初是作为概念验证开始的，您可能更想使用 Flower 替代它。

启动：

```console
celery -A proj events
```

您应该会看到类似这样的屏幕：

![Celery Events](../assets/celeryevshotsm.jpg)

`celery events` 也用于启动快照相机（参见 :ref:`monitoring-snapshots`：

```console
celery -A proj events --camera=<camera-class> --frequency=1.0
```

并且它包含一个将事件转储到 `stdout` 的工具：

```console
celery -A proj events --dump
```

要获取完整的选项列表，请使用 `--help`：

```console
celery events --help
```

## RabbitMQ

要管理 Celery 集群，了解如何监控 RabbitMQ 非常重要。

RabbitMQ 自带了 [`rabbitmqctl(1)`](https://www.rabbitmq.com/man/rabbitmqctl.1.man.html) 命令，使用它可以列出队列、交换机、绑定关系、队列长度、每个队列的内存使用情况，以及管理用户、虚拟主机及其权限。

!!! note

    这些示例中使用的是默认虚拟主机 (`"/"`)，如果您使用自定义虚拟主机，必须在命令中添加 `-p` 参数，例如：`rabbitmqctl list_queues -p my_vhost …`

### 检查队列

查找队列中的任务数量：

```console
rabbitmqctl list_queues name messages messages_ready messages_unacknowledged
```

这里 `messages_ready` 是准备投递的消息数量（已发送但未接收），`messages_unacknowledged` 是已被工作进程接收但尚未确认的消息数量（意味着正在进行中，或已被保留）。`messages` 是准备和未确认消息的总和。

查找当前从队列消费的工作进程数量：

```console
rabbitmqctl list_queues name consumers
```

查找分配给队列的内存量：

```console
rabbitmqctl list_queues name memory
```

!!! tip "向 `rabbitmqctl(1)` 添加 `-q` 选项可以使输出更易于解析。"

## Redis

如果您使用 Redis 作为代理，可以使用 `redis-cli(1)` 命令来监控 Celery 集群，列出队列的长度。

### 检查队列

查找队列中的任务数量：

```console
redis-cli -h HOST -p PORT -n DATABASE_NUMBER llen QUEUE_NAME
```

默认队列名为 `celery`。要获取所有可用的队列，请调用：

```console
redis-cli -h HOST -p PORT -n DATABASE_NUMBER keys \*
```

!!! note

    队列键仅在其中有任务时存在，因此如果某个键不存在，仅表示该队列中没有消息。这是因为在 Redis 中，没有元素的列表会自动被移除，因此它不会出现在 `keys` 命令的输出中，并且该列表的 `llen` 返回 0。

    此外，如果您将 Redis 用于其他目的，`keys` 命令的输出将包括数据库中存储的不相关值。推荐的解决方法是使用专用的 `DATABASE_NUMBER` 给 Celery，您也可以使用数据库编号来将 Celery 应用程序彼此分开（虚拟主机），但这不会影响监控事件，例如 Flower 使用的Redis pub/sub 命令是全局的，而不是基于数据库的。

## Munin {#monitoring-munin}

这是一个已知的Munin插件列表，在维护Celery集群时可能很有用。

- [rabbitmq-munin](https://github.com/ask/rabbitmq-munin): RabbitMQ的Munin插件。
- [celery_tasks](https://github.com/munin-monitoring/contrib/blob/master/plugins/celery/celery_tasks): 监控每种任务类型已被执行的次数
  （需要 `celerymon`）。
- [celery_tasks_states](https://github.com/munin-monitoring/contrib/blob/master/plugins/celery/celery_tasks_states): 监控每种状态下任务的数量
  （需要 `celerymon`）。

## 事件

工作器能够在事件发生时发送消息。这些事件随后被诸如 Flower 和 `celery events` 等工具捕获，用于监控集群。

### 快照 {#monitoring-snapshots}

即使单个工作器也能产生大量事件，因此在磁盘上存储所有事件的历史可能非常昂贵。

一系列事件描述了该时间段内的集群状态，通过定期对此状态进行快照，您可以保留所有历史记录，但仍仅定期将其写入磁盘。

要拍摄快照，您需要一个 Camera 类，通过它您可以定义每次捕获状态时应发生的情况；您可以将其写入数据库、通过电子邮件发送或完全执行其他操作。

然后使用 `celery events` 通过摄像头拍摄快照，例如，如果您想每 2 秒使用摄像头 `myapp.Camera` 捕获状态，您可以使用以下参数运行 `celery events`：

```console
celery -A proj events -c myapp.Camera --frequency=2.0
```

#### 自定义摄像头

如果您需要以一定间隔捕获事件并对这些事件执行某些操作，摄像头会很有用。对于实时事件处理，您应该直接使用 :class:`@events.Receiver`，如 :ref:`event-real-time-example` 中所示。

这是一个示例摄像头，将快照转储到屏幕：

```python
from pprint import pformat

from celery.events.snapshot import Polaroid

class DumpCam(Polaroid):
    clear_after = True  # 刷新后清除（包括 state.event_count）。

    def on_shutter(self, state):
        if not state.event_count:
            # 自上次快照以来没有新事件。
            return
        print('Workers: {0}'.format(pformat(state.workers, indent=4)))
        print('Tasks: {0}'.format(pformat(state.tasks, indent=4)))
        print('Total: {0.event_count} events, {0.task_count} tasks'.format(state))
```

有关状态对象的更多信息，请参阅 `celery.events.state` 的 API 参考。

现在您可以通过指定 `celery events -c` 选项将此摄像头与 `celery events` 一起使用：

```console
celery -A proj events -c myapp.DumpCam --frequency=2.0
```

或者您可以像这样以编程方式使用它：

```python
from celery import Celery
from myapp import DumpCam

def main(app, freq=1.0):
    state = app.events.State()
    with app.connection() as connection:
        recv = app.events.Receiver(connection, handlers={'*': state.event})
        with DumpCam(state, freq=freq):
            recv.capture(limit=None, timeout=None)

if __name__ == '__main__':
    app = Celery(broker='amqp://guest@localhost//')
    main(app)
```

### 实时处理

要实时处理事件，您需要以下内容：

- 事件消费者（这是 `Receiver`）

- 事件到达时调用的一组处理程序。

    您可以为每种事件类型设置不同的处理程序，或者可以使用通配符处理程序（'*'）

- 状态（可选）

  `State` 是集群中任务和工作器的便捷内存表示形式，随着事件的到来而更新。

  它封装了许多常见问题的解决方案，例如检查工作器是否仍然存活（通过验证心跳）、在事件到达时合并事件字段、确保时间戳同步等等。


结合这些，您可以轻松地实时处理事件：

```python
from celery import Celery


def my_monitor(app):
    state = app.events.State()

    def announce_failed_tasks(event):
        state.event(event)
        # 任务名称仅在 -received 事件中发送，状态
        # 将为我们跟踪这一点。
        task = state.tasks.get(event['uuid'])

        print('TASK FAILED: %s[%s] %s' % (
            task.name, task.uuid, task.info(),))

    with app.connection() as connection:
        recv = app.events.Receiver(connection, handlers={
                'task-failed': announce_failed_tasks,
                '*': state.event,
        })
        recv.capture(limit=None, timeout=None, wakeup=True)

if __name__ == '__main__':
    app = Celery(broker='amqp://guest@localhost//')
    my_monitor(app)
```

!!! note

    `capture` 的 `wakeup` 参数向所有工作器发送信号，强制它们发送心跳。这样，当监控器启动时，您可以立即看到工作器。

您可以通过指定处理程序来监听特定事件：

```python
from celery import Celery

def my_monitor(app):
    state = app.events.State()

    def announce_failed_tasks(event):
        state.event(event)
        # 任务名称仅在 -received 事件中发送，状态
        # 将为我们跟踪这一点。
        task = state.tasks.get(event['uuid'])

        print('TASK FAILED: %s[%s] %s' % (
            task.name, task.uuid, task.info(),))

    with app.connection() as connection:
        recv = app.events.Receiver(connection, handlers={
                'task-failed': announce_failed_tasks,
        })
        recv.capture(limit=None, timeout=None, wakeup=True)

if __name__ == '__main__':
    app = Celery(broker='amqp://guest@localhost//')
    my_monitor(app)
```

## 事件参考

此列表包含工作进程发送的事件及其参数。

### 任务事件

#### task-sent

`task-sent(uuid, name, args, kwargs, retries, eta, expires, queue, exchange, routing_key, root_id, parent_id)`

当任务消息发布且 `task_send_sent_event` 设置启用时发送。

#### task-received

`task-received(uuid, name, args, kwargs, retries, eta, hostname, timestamp, root_id, parent_id)`

当工作进程接收到任务时发送。

#### task-started

`task-started(uuid, hostname, timestamp, pid)`

在工作进程执行任务之前发送。

#### task-succeeded

`task-succeeded(uuid, result, runtime, hostname, timestamp)`

如果任务执行成功则发送。

运行时间是指使用池执行任务所需的时间。（从任务发送到工作进程池开始，到池结果处理程序回调被调用时结束）。

#### task-failed

`task-failed(uuid, exception, traceback, hostname, timestamp)`

如果任务执行失败则发送。

#### task-rejected

`task-rejected(uuid, requeue)`

任务被工作进程拒绝，可能被重新排队或移动到死信队列。

#### task-revoked

`task-revoked(uuid, terminated, signum, expired)`

如果任务已被撤销则发送（请注意，这很可能由多个工作进程发送）。

| 参数           | 描述                                         |
|--------------|--------------------------------------------|
| `terminated` | 如果任务进程被终止则设置为 true，并且 `signum` 字段设置为使用的信号。 |
| `expired`    | 如果任务已过期则设置为 true。                          |

#### task-retried

`task-retried(uuid, exception, traceback, hostname, timestamp)`

如果任务失败但将在未来重试则发送。

### 工作进程事件

#### worker-online

`worker-online(hostname, timestamp, freq, sw_ident, sw_ver, sw_sys)`

工作进程已连接到代理并在线。

| 参数          | 描述                        |
|-------------|---------------------------|
| `hostname`  | 工作进程的节点名。                 |
| `timestamp` | 事件时间戳。                    |
| `freq`      | 心跳频率（秒，浮点数）。              |
| `sw_ident`  | 工作进程软件名称（例如 `py-celery`）。 |
| `sw_ver`    | 软件版本（例如 2.2.0）。           |
| `sw_sys`    | 操作系统（例如 Linux/Darwin）。    |

#### worker-heartbeat

`worker-heartbeat(hostname, timestamp, freq, sw_ident, sw_ver, sw_sys, active, processed)`

每分钟发送一次，如果工作进程在 2 分钟内未发送心跳，则被视为离线。

| 参数          | 描述                        |
|-------------|---------------------------|
| `hostname`  | 工作进程的节点名。                 |
| `timestamp` | 事件时间戳。                    |
| `freq`      | 心跳频率（秒，浮点数）。              |
| `sw_ident`  | 工作进程软件名称（例如 `py-celery`）。 |
| `sw_ver`    | 软件版本（例如 2.2.0）。           |
| `sw_sys`    | 操作系统（例如 Linux/Darwin）。    |
| `active`    | 当前正在执行的任务数。               |
| `processed` | 此工作进程处理的总任务数。             |

#### worker-offline

`worker-offline(hostname, timestamp, freq, sw_ident, sw_ver, sw_sys)`

工作进程已与代理断开连接。

### 邮箱配置（高级）

Celery 内部使用 `kombu.pidbox.Mailbox` 向工作进程发送控制和广播命令。

高级用户可以通过自定义邮箱的创建方式来配置其行为。`Mailbox` 现在支持以下参数：

| 参数          | 描述                                       |
|-------------|------------------------------------------|
| `durable`   | 默认 `False`，如果设置为 `True`，控制交换将在代理重启后继续存在。 |
| `exclusive` | 默认 `False`，如果设置为 `True`，交换将只能由一个连接使用。    |

!!! warning

    同时设置 `durable=True` 和 `exclusive=True` 是不允许的，并且会引发错误，因为这两个选项在 AMQP 中是互不兼容的。

有关高级配置，请参阅 `event_queue_durable` 和 `event_queue_exclusive`。
