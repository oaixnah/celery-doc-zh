---
subtitle: Signals
description: Celery信号机制完整指南 - 任务、Worker、Beat、日志和事件信号详解，包含代码示例和参数说明。
---

# 信号

信号允许解耦的应用程序在应用程序的其他地方发生某些操作时接收通知。

Celery 附带了许多信号，您的应用程序可以连接到这些信号来增强某些操作的行为。

## 基础

多种类型的事件会触发信号，您可以连接到这些信号以在它们触发时执行操作。

连接到 `after_task_publish` 信号的示例：

```python
from celery.signals import after_task_publish

@after_task_publish.connect
def task_sent_handler(sender=None, headers=None, body=None, **kwargs):
    # 关于任务的信息位于任务消息的头部中
    # 使用任务协议版本 2
    info = headers if 'task' in headers else body
    print('after_task_publish for task id {info[id]}'.format(
        info=info,
    ))
```

某些信号还有一个发送者可以进行过滤。例如 `after_task_publish` 信号使用任务名称作为发送者，因此通过向 `celery.utils.dispatch.signal.Signal.connect` 提供 `sender` 参数，您可以将处理程序连接到每次发布名为 `"proj.tasks.add"` 的任务时被调用：

```python
@after_task_publish.connect(sender='proj.tasks.add')
def task_sent_handler(sender=None, headers=None, body=None, **kwargs):
    # 关于任务的信息位于任务消息的头部中
    # 使用任务协议版本 2
    info = headers if 'task' in headers else body
    print('after_task_publish for task id {info[id]}'.format(
        info=info,
    ))
```

信号使用与 `django.core.dispatch` 相同的实现。因此，其他关键字参数（例如，signal）默认会传递给所有信号处理程序。

信号处理程序的最佳实践是接受任意关键字参数（即 `**kwargs`）。这样，新的 Celery 版本可以添加额外的参数而不会破坏用户代码。

## Signals

### 任务信号

#### `before_task_publish`

在任务发布之前触发。注意：此信号在发送任务的进程中执行。

发送者是正在发送的任务名称。

| 参数             | 描述                                                                      |
|----------------|-------------------------------------------------------------------------|
| `body`         | 任务消息体。这是一个包含任务消息字段的映射。                                                  |
| `exchange`     | 要发送到的交换器名称或 `kombu.Exchange` 对象。                                        |
| `routing_key`  | 发送消息时使用的路由键。                                                            |
| `headers`      | 应用程序头部映射（可以修改）。                                                         |
| `properties`   | 消息属性（可以修改）。                                                             |
| `declare`      | 在发布消息之前要声明的实体列表（`kombu.Exchange`、`kombu.Queue` 或 `kombu.binding`）。可以修改。 |
| `retry_policy` | 重试选项的映射。可以是 `kombu.Connection.ensure` 的任何参数，并且可以修改。                     |

#### `after_task_publish`

当任务已发送到代理时触发。注意：此信号在发送任务的进程中执行。

发送者是正在发送的任务名称。

| 参数            | 描述                             |
|---------------|--------------------------------|
| `headers`     | 任务消息头部。                        |
| `body`        | 任务消息体。                         |
| `exchange`    | 使用的交换器名称或 `kombu.Exchange` 对象。 |
| `routing_key` | 使用的路由键。                        |

#### `task_prerun`

在任务执行之前触发。

发送者是正在执行的任务对象。

| 参数        | 描述                 |
|-----------|--------------------|
| `task_id` | 要执行的任务ID。          |
| `task`    | 正在执行的任务。           |
| `args`    | 任务的位置参数。           |
| `kwargs`  | 任务的关键字参数。          |

#### `task_postrun`

在任务执行之后触发。

发送者是被执行的任务对象。

| 参数        | 描述                    |
|-----------|-----------------------|
| `task_id` | 要执行的任务ID。             |
| `task`    | 正在执行的任务。              |
| `args`    | 任务的位置参数。              |
| `kwargs`  | 任务的关键字参数。             |
| `retval`  | 任务的返回值。               |
| `state`   | 结果状态的名称。              |

#### `task_retry`

当任务将要重试时触发。

发送者是任务对象。

| 参数        | 描述                                                  |
|-----------|-----------------------------------------------------|
| `request` | 当前任务请求。                                             |
| `reason`  | 重试原因（通常是一个异常实例，但总是可以强制转换为 `str`）。                   |
| `einfo`   | 详细的异常信息，包括回溯（一个 `billiard.einfo.ExceptionInfo` 对象）。 |

#### `task_success`

当任务成功时触发。

发送者是被执行的任务对象。

| 参数       | 描述               |
|----------|------------------|
| `result` | 任务的返回值。          |

#### `task_failure`

当任务失败时触发。

发送者是被执行的任务对象。

| 参数          | 描述                                 |
|-------------|------------------------------------|
| `task_id`   | 任务的ID。                             |
| `exception` | 引发的异常实例。                           |
| `args`      | 任务调用时使用的位置参数。                      |
| `kwargs`    | 任务调用时使用的关键字参数。                     |
| `traceback` | 堆栈跟踪对象。                            |
| `einfo`     | `billiard.einfo.ExceptionInfo` 实例。 |

#### `task_internal_error`

在执行任务时发生内部Celery错误时触发。

发送者是被执行的任务对象。

| 参数          | 描述                                            |
|-------------|-----------------------------------------------|
| `task_id`   | 任务的ID。                                        |
| `args`      | 任务调用时使用的位置参数。                                 |
| `kwargs`    | 任务调用时使用的关键字参数。                                |
| `request`   | 原始请求字典。提供此参数是因为在异常引发时 `task.request` 可能尚未准备好。 |
| `exception` | 引发的异常实例。                                      |
| `traceback` | 堆栈跟踪对象。                                       |
| `einfo`     | `billiard.einfo.ExceptionInfo` 实例。            |

#### `task_received`

当从代理接收到任务并准备好执行时触发。

发送者是消费者对象。

| 参数        | 描述                                                                                                                                  |
|-----------|-------------------------------------------------------------------------------------------------------------------------------------|
| `request` | 这是一个 `celery.worker.request.Request` 实例，而不是 `task.request`。在使用prefork池时，此信号在父进程中触发，因此 `task.request` 不可用且不应使用。请改用此对象，因为它们共享许多相同的字段。 |

#### `task_revoked`

当任务被工作器撤销/终止时触发。

发送者是被撤销/终止的任务对象。

| 参数           | 描述                                                                                                                            |
|--------------|-------------------------------------------------------------------------------------------------------------------------------|
| `request`    | 这是一个 `celery.app.task.Context` 实例，而不是 `task.request`。在使用prefork池时，此信号在父进程中触发，因此 `task.request` 不可用且不应使用。请改用此对象，因为它们共享许多相同的字段。 |
| `terminated` | 如果任务被终止，则设置为 `True`。                                                                                                          |
| `signum`     | 用于终止任务的信号编号。如果此值为 `None` 且 terminated 为 `True`，则应假定为 `TERM`。                                                                  |
| `expired`    | 如果任务已过期，则设置为 `True`。                                                                                                          |

#### `task_unknown`

当工作器接收到未注册任务的消息时触发。

发送者是工作器的 `celery.worker.consumer.Consumer`。

| 参数        | 描述            |
|-----------|---------------|
| `name`    | 注册表中未找到的任务名称。 |
| `id`      | 消息中找到的任务ID。   |
| `message` | 原始消息对象。       |
| `exc`     | 发生的错误。        |

#### `task_rejected`

当工作器在其任务队列之一接收到未知类型的消息时触发。

发送者是工作器的 `celery.worker.consumer.Consumer`。

| 参数        | 描述          |
|-----------|-------------|
| `message` | 原始消息对象。     | 
| `exc`     | 发生的错误（如果有）。 |

### 应用信号

#### `import_modules`

当程序（worker、beat、shell等）请求导入 `include` 和 `imports` 设置中的模块时发送此信号。

发送者是应用实例。

### Worker 信号

#### `celeryd_after_setup`

此信号在 worker 实例设置完成后但在调用 run 之前发送。这意味着来自 `celery worker -Q` 选项的任何队列都已启用，日志记录已设置等等。

它可以用于添加应始终从中消费的自定义队列，忽略 `celery worker -Q` 选项。以下是一个为每个 worker 设置直接队列的示例，这些队列随后可用于将任务路由到任何特定的 worker：

```python
from celery.signals import celeryd_after_setup

@celeryd_after_setup.connect
def setup_direct_queue(sender, instance, **kwargs):
    queue_name = '{0}.dq'.format(sender)  # sender is the nodename of the worker
    instance.app.amqp.queues.select_add(queue_name)
```

提供的参数：

| 参数         | 描述                                                                                                                  |
|------------|---------------------------------------------------------------------------------------------------------------------|
| `sender`   | Worker 的节点名称。                                                                                                       |
| `instance` | 这是要初始化的 `celery.apps.worker.Worker` 实例。请注意，到目前为止只设置了 `app` 和 `hostname`（节点名）属性，`__init__` 的其余部分尚未执行。                |
| `conf`     | 当前应用的配置。                                                                                                            |

#### `celeryd_init`

这是 `celery worker` 启动时发送的第一个信号。`sender` 是 worker 的主机名，因此此信号可用于设置特定于 worker 的配置：

```python
from celery.signals import celeryd_init

@celeryd_init.connect(sender='worker12@example.com')
def configure_worker12(conf=None, **kwargs):
    conf.task_default_rate_limit = '10/m'
```

或者要为多个 worker 设置配置，您可以在连接时省略指定 sender：

```python
from celery.signals import celeryd_init

@celeryd_init.connect
def configure_workers(sender=None, conf=None, **kwargs):
    if sender in ('worker1@example.com', 'worker2@example.com'):
        conf.task_default_rate_limit = '10/m'
    if sender == 'worker3@example.com':
        conf.worker_prefetch_multiplier = 0
```

提供的参数：

| 参数         | 描述                                                                                                   |
|------------|------------------------------------------------------------------------------------------------------|
| `sender`   | Worker 的节点名称。                                                                                        |
| `instance` | 这是要初始化的 `celery.apps.worker.Worker` 实例。请注意，到目前为止只设置了 `app` 和 `hostname`（节点名）属性，`__init__` 的其余部分尚未执行。 |
| `conf`     | 当前应用的配置。                                                                                             |
| `options`  | 从命令行参数（包括默认值）传递给 worker 的选项。                                                                         |

#### `worker_init`

在 worker 启动之前分发。

#### `worker_before_create_process`

在父进程中分发，就在 prefork 池中创建新的子进程之前。它可以用于清理在 fork 时行为不佳的实例。

```python
@signals.worker_before_create_process.connect
def clean_channels(**kwargs):
    grpc_singleton.clean_channel()
```

#### `worker_ready`

当 worker 准备好接受工作时分发。

#### `heartbeat_sent`

当 Celery 发送 worker 心跳时分发。

发送者是 `celery.worker.heartbeat.Heart` 实例。

#### `worker_shutting_down`

当 worker 开始关闭过程时分发。

提供的参数：

| 参数         | 描述                |
|------------|-------------------|
| `sig`      | 接收到的 POSIX 信号。    |
| `how`      | 关闭方法，warm 或 cold。 |
| `exitcode` | 主进程退出时将使用的退出代码。   |

#### `worker_process_init`

在所有池子进程启动时分发。

请注意，附加到此信号的处理程序阻塞时间不得超过 4 秒，否则进程将被杀死，假设启动失败。

#### `worker_process_shutdown`

在所有池子进程退出之前分发。

注意：不能保证此信号会被分发，类似于 `finally` 块，无法保证在关闭时会调用处理程序，如果被调用，可能会在过程中被中断。

提供的参数：

| 参数         | 描述              |
|------------|-----------------|
| `pid`      | 即将关闭的子进程的 pid。  |
| `exitcode` | 子进程退出时将使用的退出代码。 |

#### `worker_shutdown`

当 worker 即将关闭时分发。

### Beat 信号

#### `beat_init`

当 `celery beat` 启动时（独立运行或嵌入式）发送。

发送者是 `celery.beat.Service` 实例。

#### `beat_embedded_init`

当 `celery beat` 作为嵌入式进程启动时，除了 `beat_init` 信号外还会发送此信号。

发送者是 `celery.beat.Service` 实例。

### Eventlet 信号

#### `eventlet_pool_started`

当 eventlet 池启动后发送。

发送者是 `celery.concurrency.eventlet.TaskPool` 实例。

#### `eventlet_pool_preshutdown`

在 worker 关闭时发送，就在 eventlet 池被要求等待剩余 worker 之前。

发送者是 `celery.concurrency.eventlet.TaskPool` 实例。

#### `eventlet_pool_postshutdown`

当池已加入且 worker 准备关闭时发送。

发送者是 `celery.concurrency.eventlet.TaskPool` 实例。

#### `eventlet_pool_apply`

每当任务被应用到池时发送。

发送者是 `celery.concurrency.eventlet.TaskPool` 实例。

| 参数       | 描述     |
|----------|--------|
| `target` | 目标函数。  |
| `args`   | 位置参数。  |
| `kwargs` | 关键字参数。 |

### 日志信号

#### `setup_logging`

如果连接了此信号，Celery 将不会配置日志记录器，因此您可以使用此信号完全用自己的日志配置覆盖默认配置。

如果您想增强 Celery 设置的日志配置，可以使用 `after_setup_logger` 和 `after_setup_task_logger` 信号。

| 参数         | 描述          |
|------------|-------------|
| `loglevel` | 日志对象的级别。    |
| `logfile`  | 日志文件的名称。    |
| `format`   | 日志格式字符串。    |
| `colorize` | 指定日志消息是否着色。 |

#### `after_setup_logger`

在每个全局日志记录器（非任务日志记录器）设置后发送。用于增强日志配置。

| 参数         | 描述          |
|------------|-------------|
| `logger`   | 日志记录器对象。    |
| `loglevel` | 日志对象的级别。    |
| `logfile`  | 日志文件的名称。    |
| `format`   | 日志格式字符串。    |
| `colorize` | 指定日志消息是否着色。 |

#### `after_setup_task_logger`

在每个单独的任务日志记录器设置后发送。用于增强日志配置。

| 参数         | 描述          |
|------------|-------------|
| `logger`   | 日志记录器对象。    |
| `loglevel` | 日志对象的级别。    |
| `logfile`  | 日志文件的名称。    |
| `format`   | 日志格式字符串。    |
| `colorize` | 指定日志消息是否着色。 |

### 命令信号

#### `user_preload_options`

此信号在任何 Celery 命令行程序完成解析用户预加载选项后发送。

它可以用于向 `celery` 伞式命令添加额外的命令行参数：

```python
from celery import Celery
from celery import signals
from celery.bin.base import Option

app = Celery()
app.user_options['preload'].add(Option(
    '--monitoring', action='store_true',
    help='Enable our external monitoring utility, blahblah',
))

@signals.user_preload_options.connect
def handle_preload_options(options, **kwargs):
    if options['monitoring']:
        enable_monitoring()
```

发送者是 `celery.bin.base.Command` 实例，其值取决于被调用的程序（例如，对于伞式命令，它将是 `celery.bin.celery.CeleryCommand`) 对象）。

提供参数：

| 参数        | 描述                     |
|-----------|------------------------|
| `app`     | 应用实例。                  |
| `options` | 已解析的用户预加载选项的映射（包含默认值）。 |

### 已弃用信号

#### `task_sent`

此信号已弃用，请改用 `after_task_publish`。
