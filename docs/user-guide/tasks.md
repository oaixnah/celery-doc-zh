---
subtitle: Tasks
description: Celery任务系统完整指南 - 学习如何创建、配置和管理Celery任务，包括任务装饰器使用、绑定任务、任务继承、重试机制、日志记录、参数检查等核心功能。涵盖任务命名、请求上下文、敏感信息处理以及最佳实践，帮助开发者构建可靠高效的分布式任务队列系统。
---

# 任务

任务是 Celery 应用程序的构建块。

任务是一个可以从任何可调用对象创建的类。它扮演双重角色，既定义了任务被调用时会发生什么（发送消息），也定义了工作进程接收到该消息时会发生什么。

每个任务类都有一个唯一的名称，这个名称在消息中被引用，以便工作进程可以找到正确的函数来执行。

任务消息在被工作进程 `确认` 之前不会从队列中移除。工作进程可以提前保留许多消息，即使工作进程被杀死——由于电源故障或其他原因——消息也会被重新传递给另一个工作进程。

理想情况下，任务函数应该是 `幂等` 的：意味着即使使用相同的参数多次调用该函数，也不会导致意外的副作用。由于工作进程无法检测您的任务是否幂等，默认行为是在执行之前提前确认消息，这样已经启动的任务调用永远不会再次执行。

如果您的任务是幂等的，您可以设置 `acks_late` 选项，让工作进程在任务返回 *之后* 确认消息。另请参阅常见问题解答条目 [我应该使用 retry 还是 acks_late？](../faq.md#acks_late-vs-retry)。

请注意，即使启用了 `acks_late`，如果执行任务的子进程被终止（无论是由于任务调用 `sys.exit()` 还是信号），工作进程也会确认消息。这种行为是有意为之的，因为...

1. 我们不想重新运行那些迫使内核向进程发送 `SIGSEGV`（段错误）或类似信号的任务。
2. 我们假设系统管理员故意杀死任务时，不希望它自动重启。
3. 分配过多内存的任务有触发内核 OOM killer 的危险，同样的情况可能会再次发生。
4. 重新传递时总是失败的任务可能会导致高频消息循环，从而拖垮系统。

如果您确实希望在这些情况下重新传递任务，您应该考虑启用 `task_reject_on_worker_lost` 设置。

!!! warning

    无限期阻塞的任务最终可能会阻止工作进程实例执行任何其他工作。

    如果您的任务执行 I/O 操作，请确保为这些操作添加超时，例如使用 `requests` 库为 Web 请求添加超时：

    ```python
    connect_timeout, read_timeout = 5.0, 30.0
    response = requests.get(URL, timeout=(connect_timeout, read_timeout))
    ```

    [工作进程指南 - 时间限制](../user-guide/workers.md#time-limits){target="_blank"} 对于确保所有任务及时返回很方便，但时间限制事件实际上会强制杀死进程，因此仅在没有使用手动超时的情况下使用它们来检测问题。

    在以前的版本中，默认的 prefork 池调度器对长时间运行的任务不友好，因此如果您有运行几分钟/小时的任务，建议启用 `-Ofair <celery worker -O>` 命令行参数给 `celery worker`。然而，从版本 4.0 开始，-Ofair 现在是默认的调度策略。有关更多信息，请参阅 [优化指南 - 预取限制](optimizing.md#optimizing-prefetch-limit){target="_blank"}，为了获得最佳性能，请将长时间运行和短时间运行的任务路由到专用工作进程 [路由指南 - 自动路由](routing.md#routing-automatic)。

    如果您的 worker 挂起，请在提交问题之前调查正在运行的任务，因为挂起很可能是由于一个或多个任务在网络操作上挂起引起的。

## 基础

您可以通过使用 `task()` 装饰器轻松地从任何可调用对象创建任务：

```python
from .models import User

@app.task
def create_user(username, password):
    User.objects.create(username=username, password=password)
```

还有许多 [选项](#task-options) 可以为任务设置，这些可以作为装饰器的参数指定：

```python
@app.task(serializer='json')
def create_user(username, password):
    User.objects.create(username=username, password=password)
```

### 如何导入任务装饰器？

任务装饰器可在您的 `Celery()` 应用程序实例上使用，如果您不知道这是什么，请阅读 [快速上手](../getting-started/first-steps-with-celery.md)。

如果您正在使用 Django（参见 [与 Django 一起使用](../django.md)），或者您是库的作者，那么您可能想要使用 `shared_task()` 装饰器：

```python
from celery import shared_task

@shared_task
def add(x, y):
    return x + y
```

### 多个装饰器

当将多个装饰器与任务装饰器结合使用时，您必须确保 `task` 装饰器最后应用（奇怪的是，在 Python 中这意味着它必须在列表的最前面）：

```python
@app.task
@decorator2
@decorator1
def add(x, y):
    return x + y
```

### 绑定任务

任务被绑定意味着任务的第一个参数将始终是任务实例（`self`），就像 Python 的绑定方法一样：

```python
logger = get_task_logger(__name__)

@app.task(bind=True)
def add(self, x, y):
    logger.info(self.request.id)
```

绑定任务对于重试（使用 `Task.retry()`）、访问当前任务请求的信息以及您添加到自定义任务基类的任何附加功能都是必需的。

### 任务继承

任务装饰器的 `base` 参数指定任务的基类：

```python
import celery

class MyTask(celery.Task):

    def on_failure(self, exc, task_id, args, kwargs, einfo):
        print('{0!r} failed: {1!r}'.format(task_id, exc))

@app.task(base=MyTask)
def add(x, y):
    raise KeyError()
```

## 名称 {#task-names}

每个任务都必须具有唯一的名称。

如果没有提供明确的名称，任务装饰器将为您生成一个名称，该名称将基于：任务定义所在的模块，以及任务函数的名称。

设置明确名称的示例：

```pycon
>>> @app.task(name='sum-of-two-numbers')
>>> def add(x, y):
...     return x + y

>>> add.name
'sum-of-two-numbers'
```

最佳实践是使用模块名称作为命名空间，这样如果另一个模块中已经定义了同名任务，名称就不会冲突。

```pycon
>>> @app.task(name='tasks.add')
>>> def add(x, y):
...     return x + y
```

您可以通过检查任务的 `.name` 属性来了解任务的名称：

```pycon
>>> add.name
'tasks.add'
```

我们在此指定的名称（`tasks.add`）与如果任务在名为 `tasks.py` 的模块中定义时自动生成的名称完全相同：

```python title="tasks.py"
@app.task
def add(x, y):
    return x + y
```

```pycon
>>> from tasks import add
>>> add.name
'tasks.add'
```

!!! note

    您可以在 worker 中使用 `inspect` 命令查看所有已注册任务的名称。请参阅用户指南的 [监控指南](monitoring.md#inspect-registered){target="_blank"} 部分中的 `inspect registered` 命令。

### 更改自动命名行为

在某些情况下，默认的自动命名并不合适。考虑在许多不同模块中有许多任务的情况：

```console
project
├── __init__.py
├── celery.py
├── moduleA
│   ├── __init__.py
│   └── tasks.py
└── moduleB
    ├── __init__.py
    └── tasks.py
```

使用默认的自动命名，每个任务将具有类似 `moduleA.tasks.taskA`、`moduleA.tasks.taskB`、`moduleB.tasks.test` 等的生成名称。您可能希望在所有任务名称中去掉 `tasks`。如上所述，您可以为所有任务明确指定名称，或者可以通过重写 `gen_task_name()` 来更改自动命名行为。继续上面的示例，`celery.py` 可能包含：

```python
from celery import Celery

class MyCelery(Celery):

    def gen_task_name(self, name, module):
        if module.endswith('.tasks'):
            module = module[:-6]
        return super().gen_task_name(name, module)

app = MyCelery('main')
```

这样每个任务将具有类似 `moduleA.taskA`、`moduleA.taskB` 和 `moduleB.test` 的名称。

!!! warning

    请确保您的 `gen_task_name()` 是一个纯函数：意味着对于相同的输入，它必须始终返回相同的输出。

## 请求

`Task.request()` 包含与当前正在执行的任务相关的信息和状态。

请求定义了以下属性：

| 属性                    | 描述                                                                                                        |
|-----------------------|-----------------------------------------------------------------------------------------------------------|
| id                    | 执行任务的唯一ID。                                                                                                |
| group                 | 任务的 [画布指南](canvas.md#canvas-group){target="_blank"} 的唯一ID（如果此任务是成员）。                                      |
| chord                 | 此任务所属的chord的唯一ID（如果任务是header的一部分）。                                                                        |
| correlation_id        | 用于去重等用途的自定义ID。                                                                                            |
| args                  | 位置参数。                                                                                                     |
| kwargs                | 关键字参数。                                                                                                    |
| origin                | 发送此任务的主机名。                                                                                                |
| retries               | 当前任务已重试的次数。从 `0` 开始的整数。                                                                                   |
| is_eager              | 如果任务在客户端本地执行而不是由工作进程执行，则设置为 `True`。                                                                       |
| eta                   | 任务的原始ETA（如果有）。这是UTC时间（取决于 `enable_utc` 设置）。                                                               |
| expires               | 任务的原始过期时间（如果有）。这是UTC时间（取决于 `enable_utc` 设置）。                                                              |
| hostname              | 执行任务的工作进程实例的节点名称。                                                                                         |
| delivery_info         | 额外的消息传递信息。这是一个包含用于传递此任务的交换机和路由键的映射。例如 `Task.retry()` 使用它来将任务重新发送到相同的目标队列。此字典中键的可用性取决于所使用的消息代理。            |
| reply_to              | 用于发送回复的队列名称（例如，与RPC结果后端一起使用）。                                                                             |
| called_directly       | 如果任务不是由工作进程执行的，则此标志设置为true。                                                                               |
| timelimit             | 当前对此任务生效的 `(soft, hard)` 时间限制元组（如果有）。                                                                     |
| callbacks             | 如果此任务成功返回，则要调用的签名列表。                                                                                      |
| errbacks              | 如果此任务失败，则要调用的签名列表。                                                                                        |
| utc                   | 如果调用者启用了UTC，则设置为true（`enable_utc`）。                                                                       |
| headers               | 与此任务消息一起发送的消息头映射（可能为 `None`）。                                                                             |
| reply_to              | 发送回复的位置（队列名称）。                                                                                            |
| correlation_id        | 通常与任务ID相同，通常在amqp中用于跟踪回复的用途。                                                                              |
| root_id               | 此任务所属工作流中第一个任务的唯一ID（如果有）。                                                                                 |
| parent_id             | 调用此任务的任务的唯一ID（如果有）。                                                                                       |
| chain                 | 形成链的任务的反向列表（如果有）。此列表中的最后一项将是当前任务之后的下一个任务。如果使用任务协议的版本一，链任务将在 `request.callbacks` 中。                        |
| properties            | 与此任务消息一起接收的消息属性映射（可能为 `None` 或 `{}`）                                                                      | 
| replaced_task_nesting | 任务被替换的次数（如果有的话）（可能为 `0`）                                                                                  | 

### 示例

一个访问上下文中信息的示例任务是：

```python
@app.task(bind=True)
def dump_context(self, x, y):
    print('Executing task id {0.id}, args: {0.args!r} kwargs: {0.kwargs!r}'.format(
            self.request))
```

`bind` 参数意味着该函数将是一个"绑定方法"，因此您可以访问任务类型实例上的属性和方法。

## 日志记录

工作器会自动为您设置日志记录，或者您可以手动配置日志记录。

有一个特殊的日志记录器名为 "celery.task"，您可以继承此日志记录器以自动将任务名称和唯一ID作为日志的一部分。

最佳实践是在模块顶部为所有任务创建一个公共的日志记录器：

```python
from celery.utils.log import get_task_logger

logger = get_task_logger(__name__)

@app.task
def add(x, y):
    logger.info('Adding {0} + {1}'.format(x, y))
    return x + y
```

Celery 使用标准的 Python 日志记录器库，文档可以在 [`logging`](https://docs.python.org/dev/library/logging.html){target="_blank"} 找到。

您也可以使用 `print()`，因为写入标准输出/错误输出的任何内容都将重定向到日志记录系统（您可以禁用此功能，请参阅 `worker_redirect_stdouts`）。

!!! note

    如果您在任务或任务模块中的某个地方创建了日志记录器实例，工作器不会更新重定向。

    如果您想将 `sys.stdout` 和 `sys.stderr` 重定向到自定义日志记录器，您必须手动启用此功能，例如：

    ```python
    import sys

    logger = get_task_logger(__name__)

    @app.task(bind=True)
    def add(self, x, y):
        old_outs = sys.stdout, sys.stderr
        rlevel = self.app.conf.worker_redirect_stdouts_level
        try:
            self.app.log.redirect_stdouts_to_logger(logger, rlevel)
            print('Adding {0} + {1}'.format(x, y))
            return x + y
        finally:
            sys.stdout, sys.stderr = old_outs
    ```

!!! note

    如果您需要的特定 Celery 日志记录器没有发出日志，您应该检查日志记录器是否正确传播。在此示例中，启用了 "celery.app.trace" 以便发出 "succeeded in" 日志：

    ```python
    import celery
    import logging

    @celery.signals.after_setup_logger.connect
    def on_after_setup_logger(**kwargs):
        logger = logging.getLogger('celery')
        logger.propagate = True
        logger = logging.getLogger('celery.app.trace')
        logger.propagate = True
    ```

!!! note

    如果您想完全禁用 Celery 日志记录配置，请使用 `setup_logging` 信号：

    ```python
    import celery

    @celery.signals.setup_logging.connect
    def on_setup_logging(**kwargs):
        pass
    ```

### 参数检查

Celery 会在您调用任务时验证传递的参数，就像 Python 在调用普通函数时一样：

```pycon
>>> @app.task
... def add(x, y):
...     return x + y

# 使用两个参数调用任务可以正常工作：
>>> add.delay(8, 8)
<AsyncResult: f59d71ca-1549-43e0-be41-4e8821a83c0c>

# 只使用一个参数调用任务会失败：
>>> add.delay(8)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "celery/app/task.py", line 376, in delay
    return self.apply_async(args, kwargs)
  File "celery/app/task.py", line 485, in apply_async
    check_arguments(*(args or ()), **(kwargs or {}))
TypeError: add() takes exactly 2 arguments (1 given)
```

您可以通过将任务的 `typing` 属性设置为 `False` 来禁用任何任务的参数检查：

```pycon
>>> @app.task(typing=False)
... def add(x, y):
...     return x + y

# 在本地工作，但接收任务的工作器会引发错误。
>>> add.delay(8)
<AsyncResult: f59d71ca-1549-43e0-be41-4e8821a83c0c>
```

### 隐藏参数中的敏感信息

当使用 `task_protocol` 2 或更高版本（自 4.0 起默认）时，您可以使用 `argsrepr` 和 `kwargsrepr` 调用参数来覆盖位置参数和关键字参数在日志和监控事件中的表示方式：

```pycon
>>> add.apply_async((2, 3), argsrepr='(<secret-x>, <secret-y>)')

>>> charge.s(account, card='1234 5678 1234 5678').set(
...     kwargsrepr=repr({'card': '**** **** **** 5678'})
... ).delay()
```

!!! warning

    敏感信息仍然可以被任何能够从代理读取任务消息或以其他方式拦截它的人访问。

    因此，如果您的消息包含敏感信息，您可能应该加密您的消息，或者在此示例中，信用卡的实际号码可以加密存储在安全存储中，您在任务中检索并解密。

## 重试

`retry()` 可用于重新执行任务，例如在发生可恢复错误时。

当您调用 `retry()` 时，它将使用相同的任务ID发送新消息，并确保消息传递到与原始任务相同的队列。

当任务被重试时，这也会被记录为任务状态，以便您可以使用结果实例跟踪任务的进度（参见 [任务状态](#task-states)）。

以下是使用 `retry()` 的示例：

```python
@app.task(bind=True)
def send_twitter_status(self, oauth, tweet):
    try:
        twitter = Twitter(oauth)
        twitter.update_status(tweet)
    except (Twitter.FailWhaleError, Twitter.LoginError) as exc:
        raise self.retry(exc=exc)
```

!!! note

    `retry()` 调用将引发异常，因此重试之后的任何代码都不会被执行。这是 `Retry` 异常，它不被视为错误，而是作为半谓词向工作进程表示任务将被重试，以便在启用结果后端时可以存储正确的状态。

    这是正常操作，除非重试的 `throw` 参数设置为 `False`，否则总是会发生。

任务的装饰器中的 bind 参数将提供对 `self`（任务类型实例）的访问。

`exc` 参数用于传递异常信息，这些信息在日志记录和存储任务结果时使用。异常和回溯都将在任务状态中可用（如果启用了结果后端）。

如果任务有 `max_retries` 值，当超过最大重试次数时，当前异常将被重新抛出，但以下情况不会发生：

- 未提供 `exc` 参数

    在这种情况下，将引发 `MaxRetriesExceededError` 异常。

- 没有当前异常

    如果没有原始异常可以重新抛出，则将使用 `exc` 参数，因此：

    ```python
    self.retry(exc=Twitter.LoginError())
    ```

    将引发给定的 `exc` 参数。

### 使用自定义重试延迟

当任务需要重试时，它可以在重试前等待指定的时间，默认延迟由 `default_retry_delay` 属性定义。默认设置为 3 分钟。请注意，设置延迟的单位是秒（整数或浮点数）。

您还可以提供 `countdown` 参数给 `retry()` 来覆盖此默认值。

```python
@app.task(bind=True, default_retry_delay=30 * 60)  # 30分钟后重试
def add(self, x, y):
    try:
        something_raising()
    except Exception as exc:
        # 覆盖默认延迟，在1分钟后重试
        raise self.retry(exc=exc, countdown=60)
```

### 对已知异常进行自动重试

有时，您只想在引发特定异常时重试任务。

幸运的是，您可以通过在 `task()` 装饰器中使用 `autoretry_for` 参数来告诉 Celery 自动重试任务：

```python
from twitter.exceptions import FailWhaleError

@app.task(autoretry_for=(FailWhaleError,))
def refresh_timeline(user):
    return twitter.refresh_timeline(user)
```

如果您想为内部的 `retry()` 调用指定自定义参数，请将 `retry_kwargs` 参数传递给 `task()` 装饰器：

```python
@app.task(autoretry_for=(FailWhaleError,), retry_kwargs={'max_retries': 5})
def refresh_timeline(user):
    return twitter.refresh_timeline(user)
```

这提供了手动处理异常的替代方案，上面的示例与将任务体包装在 `try` ... `except` 语句中的效果相同：

```python
@app.task
def refresh_timeline(user):
    try:
        twitter.refresh_timeline(user)
    except FailWhaleError as exc:
        raise refresh_timeline.retry(exc=exc, max_retries=5)
```

如果您想对任何错误都自动重试，只需使用：

```python
@app.task(autoretry_for=(Exception,))
def x():
    ...
```

如果您的任务依赖于另一个服务，比如向API发出请求，那么使用[指数退避](https://en.wikipedia.org/wiki/Exponential_backoff)是一个好主意，以避免用您的请求淹没该服务。幸运的是，Celery的自动重试支持使这变得容易。只需指定 `retry_backoff` 参数，如下所示：

```python
from requests.exceptions import RequestException

@app.task(autoretry_for=(RequestException,), retry_backoff=True)
def x():
    ...
```

默认情况下，这种指数退避还会引入随机[抖动](https://en.wikipedia.org/wiki/Exponential_backoff#Jitter)，以避免所有任务在同一时刻运行。它还会将最大退避延迟限制为10分钟。所有这些设置都可以通过下面记录的选项进行自定义。

您还可以在基于类的任务中设置 `autoretry_for`、`max_retries`、`retry_backoff`、`retry_backoff_max` 和 `retry_jitter` 选项：

```python
class BaseTaskWithRetry(Task):
    autoretry_for = (TypeError,)
    max_retries = 5
    retry_backoff = True
    retry_backoff_max = 700
    retry_jitter = False
```

<table>
<thead>
<tr>
<th>属性</th>
<th>描述</th>
</tr>
</thead>
<tbody>
<tr>
<td>autoretry_for</td>
<td>
异常类的列表/元组。如果在任务执行期间引发这些异常中的任何一个，任务将自动重试。
默认情况下，不会自动重试任何异常。
</td>
</tr>
<tr>
<td>max_retries</td>
<td>
一个数字。放弃前的最大重试次数。值为 <code>None</code> 表示任务将永远重试。
默认情况下，此选项设置为 <code>3</code>。
</td>
</tr>
<tr>
<td>retry_backoff</td>
<td>
布尔值或数字。如果此选项设置为 <code>True</code>，自动重试将按照
<a href="https://en.wikipedia.org/wiki/Exponential_backoff">指数退避</a> 的规则延迟。第一次重试将延迟1秒，
第二次重试将延迟2秒，第三次延迟4秒，第四次延迟8秒，依此类推。
（但是，如果启用了 <code>retry_jitter</code>，此延迟值会被修改。）
如果此选项设置为数字，则用作延迟因子。例如，如果此选项设置为 <code>3</code>，
第一次重试将延迟3秒，第二次延迟6秒，第三次延迟12秒，第四次延迟24秒，依此类推。
默认情况下，此选项设置为 <code>False</code>，自动重试不会延迟。
</td>
</tr>
<tr>
<td>retry_backoff_max</td>
<td>
一个数字。如果启用了 <code>retry_backoff</code>，此选项将设置任务自动重试之间的
最大延迟（以秒为单位）。默认情况下，此选项设置为 <code>600</code>，即10分钟。
</td>
</tr>
<tr>
<td>retry_jitter</td>
<td>
布尔值。抖动用于在指数退避延迟中引入随机性，以防止队列中的所有任务
同时执行。如果此选项设置为 <code>True</code>，由 <code>retry_backoff</code>
计算的延迟值被视为最大值，实际延迟值将是零到该最大值之间的随机数。
默认情况下，此选项设置为 <code>True</code>。
</td>
</tr>
<tr>
<td>dont_autoretry_for</td>
<td>
异常类的列表/元组。这些异常不会被自动重试。
这允许排除一些匹配 <code>autoretry_for</code> 但您不希望重试的异常。
</td>
</tr>
</tbody>
</table>

## 使用 Pydantic 进行参数验证

您可以通过传递 `pydantic=True` 来使用 [Pydantic](https://pydantic.oaix.tech/) 验证和转换参数，以及基于类型提示序列化结果。

!!! note

    参数验证仅涵盖任务端的参数/返回值。在使用 `delay()` 或 `apply_async()` 调用任务时，您仍然需要自己序列化参数。

例如：

```python
from pydantic import BaseModel

class ArgModel(BaseModel):
    value: int

class ReturnModel(BaseModel):
    value: str

@app.task(pydantic=True)
def x(arg: ArgModel) -> ReturnModel:
    # 类型提示为 Pydantic 模型的 args/kwargs 将被转换
    assert isinstance(arg, ArgModel)

    # 返回的模型将自动转换为字典
    return ReturnModel(value=f"example: {arg.value}")
```

然后可以使用与模型匹配的字典调用任务，您将收到返回的模型"转储"（使用 `BaseModel.model_dump()` 序列化）：

```pycon
>>> result = x.delay({'value': 1})
>>> result.get(timeout=1)
{'value': 'example: 1'}
```

### 联合类型，泛型参数

联合类型（例如 `Union[SomeModel, OtherModel]`）或泛型参数（例如 `list[SomeModel]`）**不**受支持。

如果您想要支持列表或类似类型，建议使用 `pydantic.RootModel`。


### 可选参数/返回值

可选参数或返回值也会得到正确处理。例如，给定以下任务：

```python
from typing import Optional

# 模型与上面相同

@app.task(pydantic=True)
def x(arg: Optional[ArgModel] = None) -> Optional[ReturnModel]:
    if arg is None:
        return None
    return ReturnModel(value=f"example: {arg.value}")
```

您将得到以下行为：

```pycon
>>> result = x.delay()
>>> result.get(timeout=1) is None
True
>>> result = x.delay({'value': 1})
>>> result.get(timeout=1)
{'value': 'example: 1'}
```

### 返回值处理

只有当返回的模型与注解匹配时，返回值才会被序列化。如果您传递不同类型的模型实例，它将*不会*被序列化。`mypy` 应该已经捕获此类错误，您应该修复类型提示。

### Pydantic 参数

还有几个影响 Pydantic 行为的选项：

<table>
<thead>
<tr>
<th>属性</th>
<th>描述</th>
</tr>
</thead>
<tbody>
<tr>
<td>pydantic_strict</td>
<td>
默认情况下，<a href="https://docs.pydantic.dev/dev/concepts/strict_mode/">严格模式</a>
是禁用的。您可以传递 <code>True</code> 来启用严格模型验证。
</td>
</tr>
<tr>
<td>pydantic_context</td>
<td>
在 Pydantic 模型验证期间传递<a href="https://docs.pydantic.dev/dev/concepts/validators/#validation-context">额外的验证上下文</a>。
默认情况下，上下文已经包含应用程序对象作为 <code>celery_app</code> 和任务名称作为 <code>celery_task_name</code>。
</td>
</tr>
<tr>
<td>pydantic_dump_kwargs</td>
<td>
在序列化结果时，将这些附加参数传递给 <code>dump_kwargs()</code>。
默认情况下，只传递 <code>mode='json'</code>。
</td>
</tr>
</tbody>
</table>

## 选项列表 {#task-options}

任务装饰器可以接受许多选项来改变任务的行为方式，例如您可以使用 `rate_limit` 选项设置任务的速率限制。

传递给任务装饰器的任何关键字参数实际上都将设置为结果任务类的属性，这是内置属性的列表。

### 通用选项

<table>
<thead>
<tr>
<th>属性</th>
<th>描述</th>
</tr>
</thead>
<tbody>
<tr>
<td>name</td>
<td>任务注册的名称。<br/><br/>您可以手动设置此名称，或者将使用模块和类名自动生成一个名称。</td>
</tr>
<tr>
<td>request</td>
<td>如果任务正在执行，这将包含有关当前请求的信息。使用线程本地存储。</td>
</tr>
<tr>
<td>max_retries</td>
<td>
仅当任务调用 <code>self.retry</code> 或任务使用 <code>autoretry_for</code> 参数装饰时适用。<br/><br/>
放弃前尝试重试的最大次数。<br/>
如果重试次数超过此值，将引发 <code>MaxRetriesExceededError</code> 异常。<br/><br/>
默认值为 <code>3</code>。值为 <code>None</code> 将禁用重试限制，任务将无限重试直到成功。
</td>
</tr>
<tr>
<td>throws</td>
<td>
可选的预期错误类元组，这些错误不应被视为实际错误。<br/><br/>
此列表中的错误将被报告为结果后端的失败，但工作进程不会将该事件记录为错误，并且不会包含回溯信息。<br/><br/>
示例：

```python
@task(throws=(KeyError, HttpNotFound)):
def get_foo():
    something()
```

错误类型：<br/><br/>

- 预期错误（在 `Task.throws` 中）：以 `INFO` 严重性记录，排除回溯信息。<br/>
- 意外错误：以 `ERROR` 严重性记录，包含回溯信息。
</td>
</tr>
<tr>
<td>default_retry_delay</td>
<td>任务重试前默认等待时间（以秒为单位）。可以是 <code>int</code> 或 <code>float</code>。默认是三分钟的延迟。</td>
</tr>
<tr>
<td>rate_limit</td>
<td>
设置此任务类型的速率限制（限制在给定时间范围内可以运行的任务数量）。当速率限制生效时，任务仍将完成，但可能需要一些时间才能允许开始。<br/><br/>
如果为 <code>None</code>，则没有速率限制生效。如果是整数或浮点数，则解释为"每秒任务数"。<br/><br/>
可以通过在值后附加 <code>"/s"</code>、<code>"/m"</code> 或 <code>"/h"</code> 来指定秒、分钟或小时的速率限制。任务将在指定的时间范围内均匀分布。<br/><br/>
示例：<code>"100/m"</code>（每分钟一百个任务）。这将强制在同一工作实例上启动两个任务之间至少有600毫秒的延迟。<br/><br/>
默认是 <code>task_default_rate_limit</code> 设置：如果未指定，意味着默认情况下任务速率限制被禁用。<br/><br/>
请注意，这是*每个工作实例*的速率限制，而不是全局速率限制。要强制执行全局速率限制（例如，对于具有每秒最大请求数的API），您必须限制到特定队列。
</td>
</tr>
<tr>
<td>time_limit</td>
<td>此任务的硬时间限制（以秒为单位）。未设置时使用工作进程的默认值。</td>
</tr>
<tr>
<td>soft_time_limit</td>
<td>此任务的软时间限制。未设置时使用工作进程的默认值。</td>
</tr>
<tr>
<td>ignore_result</td>
<td>
不存储任务状态。请注意，这意味着您不能使用 <code>AsyncResult</code> 来检查任务是否准备就绪或获取其返回值。<br/><br/>
注意：如果禁用任务结果，某些功能将无法工作。有关更多详细信息，请查看 Canvas 文档。
</td>
</tr>
<tr>
<td>store_errors_even_if_ignored</td>
<td>如果为 <code>True</code>，即使任务配置为忽略结果，错误也将被存储。</td>
</tr>
<tr>
<td>serializer</td>
<td>
标识要使用的默认序列化方法的字符串。默认为 <code>task_serializer</code> 设置。可以是 <code>pickle</code>、<code>json</code>、<code>yaml</code>，或任何已在 <code>kombu.serialization.registry</code> 中注册的自定义序列化方法。<br/><br/>
请参阅 <a href="/user-guide/calling/#calling-serializers">调用指南 - 序列化器</a> 获取更多信息。
</td>
</tr>
<tr>
<td>compression</td>
<td>
标识要使用的默认压缩方案的字符串。<br/><br/>
默认为 <code>task_compression</code> 设置。可以是 <code>gzip</code>、<code>bzip2</code>，或任何已在 <code>kombu.compression</code> 注册表中注册的自定义压缩方案。<br/><br/>
请参阅 <a href="/user-guide/calling/#calling-compression">调用指南 - 压缩</a> 获取更多信息。
</td>
</tr>
<tr>
<td>backend</td>
<td>用于此任务的结果存储后端。<code>celery.backends</code> 中后端类之一的实例。默认为 <code>app.backend</code>，由 <code>result_backend</code> 设置定义。</td>
</tr>
<tr>
<td>acks_late</td>
<td>
如果设置为 <code>True</code>，此任务的消息将在任务执行**后**确认，而不是*刚好在执行前*（默认行为）。<br/><br/>
注意：这意味着如果工作进程在执行过程中崩溃，任务可能会被执行多次。请确保您的任务是 幂等的。<br/><br/>
全局默认值可以通过 <code>task_acks_late</code> 设置覆盖。
</td>
</tr>
<tr>
<td>track_started</td>
<td>
如果为 <code>True</code>，当任务由工作进程执行时，任务将报告其状态为 "started"。默认值为 <code>False</code>，
因为正常行为是不报告该粒度级别。任务要么是挂起、完成，要么是等待重试。拥有"started"状态对于长时间运行的任务很有用，需要报告当前正在运行的任务。<br/><br/>
执行任务的工作进程的主机名和进程ID将在状态元数据中可用（例如，`result.info['pid']`）<br/><br/>
全局默认值可以通过 `task_track_started` 设置覆盖。
</td>
</tr>
</tbody>
</table>

## 状态 {#task-states}

Celery 可以跟踪任务的当前状态。状态还包含成功任务的结果，或失败任务的异常和回溯信息。

有多个*结果后端*可供选择，它们都有不同的优势和劣势。

在任务的生命周期中，它会经历多个可能的状态，每个状态都可能附加任意的元数据。当任务进入新状态时，之前的状态会被遗忘，但某些转换可以被推断出来（例如，现在处于 `FAILED` 状态的任务，意味着它曾经处于 `STARTED` 状态）。

还有状态集合，如 `FAILURE_STATES` 集合和 `READY_STATES` 集合。

客户端使用这些集合的成员关系来决定是否应重新引发异常（`PROPAGATE_STATES`），或者状态是否可以缓存（如果任务准备就绪，则可以缓存）。

### 结果后端

如果您想跟踪任务或需要返回值，那么 Celery 必须将状态存储或发送到某个地方，以便以后可以检索。有多个内置的结果后端可供选择：SQLAlchemy/Django ORM、Memcached、RabbitMQ/QPid（`rpc`）和 Redis——或者您可以定义自己的后端。

没有哪个后端适用于所有用例。您应该阅读每个后端的优势和劣势，并选择最适合您需求的后端。

!!! warning

    后端使用资源来存储和传输结果。为确保
    资源被释放，您必须最终调用
    `get()` 或 `forget()` 在
    调用任务后返回的每个 `AsyncResult` 实例上。

#### RPC 结果后端 (RabbitMQ/QPid)

RPC 结果后端（`rpc://`）很特殊，因为它实际上并不*存储*状态，而是将它们作为消息发送。这是一个重要的区别，意味着结果*只能被检索一次*，并且*只能由发起任务的客户端*检索。两个不同的进程不能等待同一个结果。

即使有这种限制，如果您需要实时接收状态更改，它也是一个绝佳的选择。使用消息传递意味着客户端不必轮询新状态。

默认情况下，消息是瞬态的（非持久的），因此如果代理重启，结果将消失。您可以使用 `result_persistent` 设置来配置结果后端发送持久消息。

#### 数据库结果后端

将状态保存在数据库中可能对许多人很方便，特别是对于已经拥有数据库的 Web 应用程序，但它也有局限性。

* 轮询数据库以获取新状态是昂贵的，因此您应该增加操作的轮询间隔，例如 `result.get()`。

* 某些数据库使用的默认事务隔离级别不适合轮询表的更改。

    在 MySQL 中，默认事务隔离级别是 `REPEATABLE-READ`：意味着事务在提交之前不会看到其他事务所做的更改。

    建议将其更改为 `READ-COMMITTED` 隔离级别。

### 内置状态 {#task-states}

#### PENDING

任务正在等待执行或未知。任何未知的任务 ID 都被视为处于挂起状态。

#### STARTED

任务已启动。默认情况下不报告，要启用请参见 `track_started`。

| 标签        | 描述                                     |
|-----------|----------------------------------------|
| meta-data | 执行任务的 worker 进程的 `pid` 和 `hostname`。   |

#### SUCCESS

任务已成功执行。

| 标签         | 描述                  |
|------------|---------------------|
| meta-data  | `result` 包含任务的返回值。  |
| propagates | 是                   |
| ready      | 是                   |

#### FAILURE

任务执行失败。

| 标签         | 描述                                           |
|------------|----------------------------------------------|
| meta-data  | `result` 包含发生的异常，`traceback` 包含引发异常时堆栈的回溯信息。 |
| propagates | 是                                            |
| ready      | 是                                            |

#### RETRY

| 标签         | 描述                                             |
|------------|------------------------------------------------|
| meta-data  | `result` 包含导致重试的异常，`traceback` 包含引发异常时堆栈的回溯信息。 |
| propagates | 否                                              |

#### REVOKED

任务已被撤销。

| 标签         | 描述               |
|------------|------------------|
| propagates | 是                |

### 自定义状态

您可以轻松定义自己的状态，只需要一个唯一的名称。状态的名称通常是一个大写字符串。例如，您可以查看 `celery.contrib.abortable` 它定义了一个自定义的 `ABORTED` 状态。

使用 `update_state()` 来更新任务的状态：

```python
@app.task(bind=True)
def upload_files(self, filenames):
    for i, file in enumerate(filenames):
        if not self.request.called_directly:
            self.update_state(state='PROGRESS',
                meta={'current': i, 'total': len(filenames)})
```

这里我创建了状态 `"PROGRESS"`，告诉任何了解此状态的应用程序该任务当前正在进行中，并且通过 `current` 和 `total` 计数作为状态元数据的一部分，还可以知道它在进程中的位置。这可以用于创建进度条等。

### 创建可序列化的异常

一个鲜为人知的 Python 事实是，异常必须符合一些简单的规则才能支持被 pickle 模块序列化。

当使用 Pickle 作为序列化器时，引发不可序列化异常的任务将无法正常工作。

为确保您的异常可序列化，异常*必须*在其 `.args` 属性中提供它被实例化时使用的原始参数。确保这一点的最简单方法是让异常调用 `Exception.__init__`。

让我们看一些有效的例子，以及一个无效的例子：

```python
# 正确：
class HttpError(Exception):
    pass

# 错误：
class HttpError(Exception):

    def __init__(self, status_code):
        self.status_code = status_code

# 正确：
class HttpError(Exception):

    def __init__(self, status_code):
        self.status_code = status_code
        Exception.__init__(self, status_code)  # <-- 必需
```

所以规则是：对于任何支持自定义参数 `*args` 的异常，必须使用 `Exception.__init__(self, *args)`。

没有对*关键字参数*的特殊支持，因此如果您希望在异常被反序列化时保留关键字参数，您必须将它们作为常规参数传递：

```python
class HttpError(Exception):

    def __init__(self, status_code, headers=None, body=None):
        self.status_code = status_code
        self.headers = headers
        self.body = body

        super(HttpError, self).__init__(status_code, headers, body)
```

## 半谓词

工作器将任务包装在一个跟踪函数中，该函数记录任务的最终状态。有许多异常可以用来向此函数发出信号，以改变它处理任务返回的方式。

### Ignore

任务可以抛出 `Ignore` 异常来强制工作器忽略该任务。这意味着不会记录该任务的状态，但消息仍然会被确认（从队列中移除）。

如果您想要实现自定义的撤销类功能，或者手动存储任务结果，可以使用此功能。

示例：在Redis集合中保留已撤销的任务：

```python
from celery.exceptions import Ignore

@app.task(bind=True)
def some_task(self):
    if redis.ismember('tasks.revoked', self.request.id):
        raise Ignore()
```

示例：手动存储结果：

```python
from celery import states
from celery.exceptions import Ignore

@app.task(bind=True)
def get_tweets(self, user):
    timeline = twitter.get_timeline(user)
    if not self.request.called_directly:
        self.update_state(state=states.SUCCESS, meta=timeline)
    raise Ignore()
```

### Reject

任务可以抛出 `Reject` 异常来使用AMQP的 `basic_reject` 方法拒绝任务消息。除非启用了 `acks_late`，否则这不会产生任何效果。

拒绝消息与确认消息具有相同的效果，但一些代理可能实现可以使用的附加功能。例如，RabbitMQ支持[死信交换](https://www.rabbitmq.com/docs/dlx)的概念，其中可以配置队列使用死信交换，被拒绝的消息会被重新投递到该交换。

Reject也可以用于重新排队消息，但请非常小心地使用此功能，因为它很容易导致无限消息循环。

示例：当任务导致内存不足条件时使用reject：

```python
import errno
from celery.exceptions import Reject

@app.task(bind=True, acks_late=True)
def render_scene(self, path):
    file = get_file(path)
    try:
        renderer.render_scene(file)

    # 如果文件太大无法放入内存
    # 我们拒绝它，以便它被重新投递到死信交换
    # 我们可以手动检查情况。
    except MemoryError as exc:
        raise Reject(exc, requeue=False)
    except OSError as exc:
        if exc.errno == errno.ENOMEM:
            raise Reject(exc, requeue=False)

    # 对于任何其他错误，我们在10秒后重试。
    except Exception as exc:
        raise self.retry(exc, countdown=10)
```

示例：重新排队消息：

```python
from celery.exceptions import Reject

@app.task(bind=True, acks_late=True)
def requeues(self):
    if not self.request.delivery_info['redelivered']:
        raise Reject('no reason', requeue=True)
    print('received two times')
```

有关 `basic_reject` 方法的更多详细信息，请查阅您的代理文档。

### Retry

`Retry` 异常由 `Task.retry` 方法抛出，用于告诉工作器任务正在被重试。

## 自定义任务类

所有任务都继承自 `Task` 类。`run()` 方法成为任务的主体。

例如，以下代码：

```python
@app.task
def add(x, y):
    return x + y
```

在幕后大致会这样做：

```python
class _AddTask(app.Task):

    def run(self, x, y):
        return x + y

    
add = app.tasks[_AddTask.name]
```

### 实例化

任务**不会**为每个请求实例化，而是在任务注册表中作为全局实例注册。

这意味着 `__init__` 构造函数在每个进程中只会被调用一次，并且任务类在语义上更接近于 Actor 模式。

如果你有一个任务：

```python
from celery import Task

class NaiveAuthenticateServer(Task):

    def __init__(self):
        self.users = {'george': 'password'}

    def run(self, username, password):
        try:
            return self.users[username] == password
        except KeyError:
            return False
```

并且你将每个请求路由到同一个进程，那么它将在请求之间保持状态。

这对于缓存资源也很有用，例如，一个缓存数据库连接的基础任务类：

```python
from celery import Task

class DatabaseTask(Task):
    _db = None

    @property
    def db(self):
        if self._db is None:
            self._db = Database.connect()
        return self._db
```

#### 每个任务使用

上述内容可以像这样添加到每个任务中：

```python
from celery.app import task

@app.task(base=DatabaseTask, bind=True)
def process_rows(self: task):
    for row in self.db.table.all():
        process_row(row)
```

`process_rows` 任务的 `db` 属性将在每个进程中始终保持相同。

#### 应用程序范围使用

你也可以在整个 Celery 应用程序中使用你的自定义类，通过在实例化应用程序时将其作为 `task_cls` 参数传递。这个参数应该是一个字符串，给出你的 Task 类的 Python 路径，或者是类本身：

```python
from celery import Celery

app = Celery('tasks', task_cls='your.module.path:DatabaseTask')
```

这将使你在应用程序中使用装饰器语法声明的所有任务都使用你的 `DatabaseTask` 类，并且都将具有 `db` 属性。

默认值是 Celery 提供的类：`'celery.app.task:Task'`。

### 处理器

任务处理器是在任务生命周期特定点执行的方法。所有处理器都在执行任务的**同一个**工作进程和线程中**同步**运行。

#### 执行时间线

以下图表显示了确切的执行顺序：

```text
Worker Process Timeline
┌───────────────────────────────────────────────────────────────┐
│  1. before_start()      ← Blocks until complete               │
│  2. run()               ← Your task function                  │
│  3. [Result Backend]    ← State + return value persisted      │
│  4. on_success() OR     ← Outcome-specific handler            │
│     on_retry() OR       │                                     │
│     on_failure()        │                                     │
│  5. after_return()      ← Always runs last                    │
└───────────────────────────────────────────────────────────────┘
```

!!! tip
   
    **关键点：**
   
    - 所有处理器都在与你的任务**相同的工作进程**中运行
    - `before_start` **阻塞**任务 - `run()` 直到它完成后才会开始
    - 结果后端在 `on_success`/`on_failure` **之前**更新 - 其他客户端可以在处理器仍在运行时看到任务已完成
    - `after_return` **总是**执行，无论任务结果如何

#### 可用的处理器

**before_start(self, task_id, args, kwargs)**

由工作进程在任务开始执行之前运行。

!!! note

    此处理器**阻塞**任务：`run()` 方法在 `before_start` 返回之前*不会*开始。

| 参数      | 描述              |
|---------|-----------------|
| task_id | 要执行的任务的唯一ID。    |
| args    | 要执行的任务的原始参数。    |
| kwargs  | 要执行的任务的原始关键字参数。 |

此处理器的返回值被忽略。

**on_success(self, retval, task_id, args, kwargs)**

成功处理器。

如果任务成功执行，由工作进程运行。

!!! note

    在任务结果已经持久化到结果后端**之后**调用。外部客户端可能在此处理器仍在运行时观察到任务为 `SUCCESS` 状态。

| 参数      | 描述             |
|---------|----------------|
| retval  | 任务的返回值。        |
| task_id | 已执行任务的唯一ID。    |
| args    | 已执行任务的原始参数。    |
| kwargs  | 已执行任务的原始关键字参数。 |

此处理器的返回值被忽略。

**on_retry(self, exc, task_id, args, kwargs, einfo)**

重试处理器。

当任务要重试时由工作进程运行。

!!! note

    在任务状态已在结果后端更新为 `RETRY` **之后**调用，但**在**重试被调度之前。

| 参数      | 描述                                      |
|---------|-----------------------------------------|
| exc     | 发送到 `retry()` 的异常。                      |
| task_id | 要重试的任务的唯一ID。                            |
| args    | 要重试的任务的原始参数。                            |
| kwargs  | 要重试的任务的原始关键字参数。                         |
| einfo   | `billiard.einfo.ExceptionInfo` 实例。      |

此处理器的返回值被忽略。

**on_failure(self, exc, task_id, args, kwargs, einfo)**

失败处理器。

当任务失败时由工作进程运行。

!!! note
   
    在任务结果已经以 `FAILURE` 状态持久化到结果后端**之后**调用。外部客户端可能在此处理器仍在运行时观察到任务失败。

| 参数      | 描述                                        |
|---------|-------------------------------------------|
| exc     | 任务引发的异常。                                  |
| task_id | 失败任务的唯一ID。                                |
| args    | 失败任务的原始参数。                                |
| kwargs  | 失败任务的原始关键字参数。                             |
| einfo   | `billiard.einfo.ExceptionInfo` 实例。        |

此处理器的返回值被忽略。

**after_return(self, status, retval, task_id, args, kwargs, einfo)**

任务返回后调用的处理器。

!!! note

    在 `on_success`/`on_retry`/`on_failure` **之后**执行。这是任务生命周期中的最终钩子，**总是**运行，无论结果如何。

| 参数      | 描述                                        |
|---------|-------------------------------------------|
| status  | 当前任务状态。                                   |
| retval  | 任务返回值/异常。                                 |
| task_id | 任务的唯一ID。                                  |
| args    | 返回的任务的原始参数。                               |
| kwargs  | 返回的任务的原始关键字参数。                            |
| einfo   | `billiard.einfo.ExceptionInfo` 实例。        |

此处理器的返回值被忽略。

### 示例用法

```python
import time
from celery import Task

class MyTask(Task):
    
    def before_start(self, task_id, args, kwargs):
        print(f"Task {task_id} starting with args {args}")
        # 这会阻塞 - run() 在此返回之前不会开始
        
    def on_success(self, retval, task_id, args, kwargs):
        print(f"Task {task_id} succeeded with result: {retval}")
        # 此时结果已经对客户端可见
        
    def on_failure(self, exc, task_id, args, kwargs, einfo):
        print(f"Task {task_id} failed: {exc}")
        # 任务状态在后端已经是 FAILURE
        
    def after_return(self, status, retval, task_id, args, kwargs, einfo):
        print(f"Task {task_id} finished with status: {status}")
        # 总是最后运行

@app.task(base=MyTask)
def my_task(x, y):
    return x + y
```

### 请求和自定义请求

当接收到运行任务的消息时，[工作进程指南](workers.md)创建一个 `request` 来表示这种需求。

自定义任务类可以通过更改属性 `celery.app.task.Task.Request` 来覆盖要使用的请求类。你可以分配自定义请求类本身，或者其完全限定名称。

请求有几个职责。自定义请求类应该覆盖所有这些职责——它们负责实际运行和跟踪任务。我们强烈建议从 `celery.worker.request.Request` 继承。

当使用 [工作进程指南 - 并发性](workers.md#worker-concurrency) 时，方法
`celery.worker.request.Request.on_timeout` 和
`celery.worker.request.Request.on_failure` 在主工作进程中执行。应用程序可以利用这种机制来检测使用 `celery.app.task.Task.on_failure` 无法检测到的故障。

例如，以下自定义请求检测并记录硬时间限制和其他故障。

```python
import logging
from celery import Task
from celery.worker.request import Request

logger = logging.getLogger('my.package')

class MyRequest(Request):
   '一个最小的自定义请求，用于记录故障和硬时间限制。'

   def on_timeout(self, soft, timeout):
       super(MyRequest, self).on_timeout(soft, timeout)
       if not soft:
          logger.warning(
              'A hard timeout was enforced for task %s',
              self.task.name
          )

   def on_failure(self, exc_info, send_failed_event=True, return_ok=False):
       super().on_failure(
           exc_info,
           send_failed_event=send_failed_event,
           return_ok=return_ok
       )
       logger.warning(
           'Failure detected for task %s',
           self.task.name
       )

class MyTask(Task):
   Request = MyRequest  # 你可以使用完全限定名称 'my.package:MyRequest'

@app.task(base=MyTask)
def some_longrunning_task():
   # 发挥你的想象力
```

## 工作原理

接下来是技术细节。这部分内容不是您必须了解的，但您可能会感兴趣。

所有定义的任务都列在一个注册表中。该注册表包含task名称及其任务类的列表。您可以自己检查这个注册表：

```pycon
>>> from proj.celery import app
>>> app.tasks
{'celery.chord_unlock':
    <@task: celery.chord_unlock>,
 'celery.backend_cleanup':
    <@task: celery.backend_cleanup>,
 'celery.chord':
    <@task: celery.chord>}
```

这是Celery内置的任务列表。请注意，任务只有在定义它们的模块被导入时才会注册。

默认加载器会导入 `imports` 设置中列出的任何模块。

`task()` 装饰器负责将您的任务注册到应用程序的任务注册表中。

当任务被发送时，不会发送实际的函数代码，只发送要执行的task名称。当工作器接收到消息时，它可以在其任务注册表中查找该名称以找到执行代码。

这意味着您的工作器应该始终与客户端使用相同的软件进行更新。这是一个缺点，但替代方案是一个尚未解决的技术挑战。

## 提示和最佳实践

### 忽略不需要的结果

如果您不关心任务的结果，请务必设置 `ignore_result` 选项，因为存储结果会浪费时间和资源。

```python
@app.task(ignore_result=True)
def mytask():
    something()
```

甚至可以使用 `task_ignore_result` 设置全局禁用结果。

在调用 `apply_async` 时，可以通过传递 `ignore_result` 布尔参数，按每次执行启用/禁用结果。

```python
@app.task
def mytask(x, y):
    return x + y

# 不会存储结果
result = mytask.apply_async((1, 2), ignore_result=True)
print(result.get()) # -> None

# 会存储结果
result = mytask.apply_async((1, 2), ignore_result=False)
print(result.get()) # -> 3
```

默认情况下，当配置了结果后端时，任务将*不会忽略结果*（`ignore_result=False`）。

选项优先级顺序如下：

1. 全局 `task_ignore_result`
2. `ignore_result` 选项
3. 任务执行选项 `ignore_result`

### 更多优化提示

您可以在 [优化指南](optimizing.md){target="_blank"} 中找到其他优化提示。

### 避免启动同步子任务

让一个任务等待另一个任务的结果是非常低效的，如果工作池耗尽，甚至可能导致死锁。

请改为使用异步设计，例如使用*回调*。

**不好的做法**：

```python
@app.task
def update_page_info(url):
    page = fetch_page.delay(url).get()
    info = parse_page.delay(page).get()
    store_page_info.delay(url, info)

@app.task
def fetch_page(url):
    return myhttplib.get(url)

@app.task
def parse_page(page):
    return myparser.parse_document(page)

@app.task
def store_page_info(url, info):
    return PageInfo.objects.create(url, info)
```

**好的做法**：

```python
def update_page_info(url):
    # fetch_page -> parse_page -> store_page
    chain = fetch_page.s(url) | parse_page.s() | store_page_info.s(url)
    chain()

@app.task()
def fetch_page(url):
    return myhttplib.get(url)

@app.task()
def parse_page(page):
    return myparser.parse_document(page)

@app.task(ignore_result=True)
def store_page_info(info, url):
    PageInfo.objects.create(url=url, info=info)
```

这里我通过链接不同的 `signature` 创建了一个任务链。您可以在 [设计工作流](designing-workflows.md){target="_blank"} 中阅读有关链和其他强大构造的内容。

默认情况下，Celery 不允许您在任务内同步运行子任务，但在罕见或极端情况下，您可能需要这样做。

**警告**：启用子任务同步运行是不推荐的！

```python
@app.task
def update_page_info(url):
    page = fetch_page.delay(url).get(disable_sync_subtasks=False)
    info = parse_page.delay(page).get(disable_sync_subtasks=False)
    store_page_info.delay(url, info)

@app.task
def fetch_page(url):
    return myhttplib.get(url)

@app.task
def parse_page(page):
    return myparser.parse_document(page)

@app.task
def store_page_info(url, info):
    return PageInfo.objects.create(url, info)
```

## 性能和策略

### 粒度

任务粒度是每个子任务所需的计算量。通常来说，将问题拆分成许多小任务比使用几个长时间运行的任务更好。

使用较小的任务，您可以并行处理更多任务，并且任务不会运行太长时间而阻塞工作进程处理其他等待的任务。

然而，执行任务确实有开销。需要发送消息，数据可能不是本地的，等等。因此，如果任务过于细粒度，增加的开销可能会抵消任何好处。

!!! quote "[《并发编程的艺术》](http://oreilly.com/catalog/9780596521547)这本书有一个专门讨论任务粒度主题的章节[AOC1][^1]。"

[^1]: Breshears, Clay. 第2.2.1节，"并发编程的艺术"。O'Reilly Media, Inc. 2009年5月15日。ISBN-13 978-0-596-52153-0。

### 数据局部性

处理任务的工作进程应尽可能靠近数据。最好是在内存中有一个副本，最坏的情况是从另一个大陆进行完整传输。

如果数据很远，您可以尝试在位置运行另一个工作进程，或者如果不可能的话，缓存经常使用的数据，或预加载您知道将要使用的数据。

在工作进程之间共享数据的最简单方法是使用分布式缓存系统，如[memcached](https://memcached.org/)。

!!! quote "Jim Gray的论文[分布式计算经济学](https://www.microsoft.com/en-us/research/publication/distributed-computing-economics/)是数据局部性主题的优秀介绍。"

### 状态

由于Celery是一个分布式系统，您无法知道任务将在哪个进程或哪台机器上执行。您甚至无法知道任务是否会及时运行。

古老的异步格言告诉我们"断言世界是任务的责任"。这意味着自任务请求以来，世界视图可能已经改变，因此任务有责任确保世界处于应有的状态；如果您有一个重新索引搜索引擎的任务，并且搜索引擎最多每5分钟才应重新索引一次，那么这必须是任务的责任来断言这一点，而不是调用者的责任。

另一个需要注意的问题是Django模型对象。它们不应作为参数传递给任务。几乎总是更好的是在任务运行时从数据库中重新获取对象，因为使用旧数据可能导致竞态条件。

想象以下场景：您有一篇文章和一个自动扩展其中某些缩写的任务：

```python
class Article(models.Model):
    title = models.CharField()
    body = models.TextField()

@app.task
def expand_abbreviations(article):
    article.body.replace('MyCorp', 'My Corporation')
    article.save()
```

首先，作者创建一篇文章并保存，然后作者点击一个按钮来启动缩写任务：

```pycon
>>> article = Article.objects.get(id=102)
>>> expand_abbreviations.delay(article)
```

现在，队列非常繁忙，因此任务将在2分钟后才会运行。与此同时，另一位作者对文章进行了更改，因此当任务最终运行时，文章正文会恢复到旧版本，因为任务参数中包含了旧正文。

修复竞态条件很容易，只需使用文章ID，并在任务体中重新获取文章：

```python
@app.task
def expand_abbreviations(article_id):
    article = Article.objects.get(id=article_id)
    article.body.replace('MyCorp', 'My Corporation')
    article.save()
```

```pycon
>>> expand_abbreviations.delay(article_id)
```

这种方法甚至可能有性能优势，因为发送大消息可能很昂贵。

### 数据库事务

让我们看另一个例子：

```python
from django.db import transaction
from django.http import HttpResponseRedirect

@transaction.atomic
def create_article(request):
    article = Article.objects.create()
    expand_abbreviations.delay(article.pk)
    return HttpResponseRedirect('/articles/')
```

这是一个Django视图，在数据库中创建一个文章对象，然后将主键传递给任务。它使用 `transaction.atomic` 装饰器，该装饰器将在视图返回时提交事务，或者在视图引发异常时回滚。

存在竞态条件，因为事务是原子性的。这意味着文章对象在视图函数返回响应之前不会持久化到数据库中。如果异步任务在事务提交之前开始执行，它可能会尝试查询尚不存在的文章对象。为了防止这种情况，我们需要确保在触发任务之前提交事务。

解决方案是使用 `delay_on_commit()` 代替：

```python
from django.db import transaction
from django.http import HttpResponseRedirect

@transaction.atomic
def create_article(request):
    article = Article.objects.create()
    expand_abbreviations.delay_on_commit(article.pk)
    return HttpResponseRedirect('/articles/')
```

此方法在Celery 5.4中添加。它是一个快捷方式，使用Django的 `on_commit` 回调在所有事务成功提交后启动您的Celery任务。

#### 对于Celery <5.4

如果您使用的是较旧版本的Celery，可以使用Django回调直接复制此行为，如下所示：

```python
import functools
from django.db import transaction
from django.http import HttpResponseRedirect

@transaction.atomic
def create_article(request):
    article = Article.objects.create()
    transaction.on_commit(
        functools.partial(expand_abbreviations.delay, article.pk)
    )
    return HttpResponseRedirect('/articles/')
```

!!! note

    `on_commit`在Django 1.9及以上版本中可用，如果您使用的是较早版本，则[django-transaction-hooks](https://github.com/carljm/django-transaction-hooks)库添加了对它的支持。

## 示例

让我们举一个真实世界的例子：一个需要过滤评论垃圾邮件的博客。当评论创建时，垃圾邮件过滤器在后台运行，因此用户无需等待其完成。

我有一个允许在博客文章上评论的Django博客应用程序。我将描述这个应用程序的模型/视图和任务部分。

### `blog/models.py`

评论模型如下所示：

```python
from django.db import models
from django.utils.translation import ugettext_lazy as _


class Comment(models.Model):
    name = models.CharField(_('name'), max_length=64)
    email_address = models.EmailField(_('email address'))
    homepage = models.URLField(_('home page'), blank=True, verify_exists=False)
    comment = models.TextField(_('comment'))
    pub_date = models.DateTimeField(_('Published date'), editable=False, auto_add_now=True)
    is_spam = models.BooleanField(_('spam?'), default=False, editable=False)

    class Meta:
        verbose_name = _('comment')
        verbose_name_plural = _('comments')
```

在发布评论的视图中，我首先将评论写入数据库，然后在后台启动垃圾邮件过滤器任务。

### `blog/views.py`

```python
from django import forms
from django.http import HttpResponseRedirect
from django.template.context import RequestContext
from django.shortcuts import get_object_or_404, render_to_response

from blog import tasks
from blog.models import Comment


class CommentForm(forms.ModelForm):

    class Meta:
        model = Comment


def add_comment(request, slug, template_name='comments/create.html'):
    post = get_object_or_404(Entry, slug=slug)
    remote_addr = request.META.get('REMOTE_ADDR')

    if request.method == 'post':
        form = CommentForm(request.POST, request.FILES)
        if form.is_valid():
            comment = form.save()
            # 异步检查垃圾邮件。
            tasks.spam_filter.delay(comment_id=comment.id,
                                    remote_addr=remote_addr)
            return HttpResponseRedirect(post.get_absolute_url())
    else:
        form = CommentForm()

    context = RequestContext(request, {'form': form})
    return render_to_response(template_name, context_instance=context)
```

为了过滤评论中的垃圾邮件，我使用[Akismet](http://akismet.com/faq/)，该服务用于过滤发布到免费博客平台[Wordpress](https://wordpress.com/)的评论中的垃圾邮件。[Akismet](http://akismet.com/faq/)对个人使用是免费的，但对于商业用途需要付费。您必须注册他们的服务才能获得API密钥。

为了向[Akismet](http://akismet.com/faq/)进行API调用，我使用由[Michael Foord](http://www.voidspace.org.uk/)编写的[akismet.py](http://www.voidspace.org.uk/downloads/akismet.py)库。

### `blog/tasks.py`

```python
from celery import Celery

from akismet import Akismet

from django.core.exceptions import ImproperlyConfigured
from django.contrib.sites.models import Site

from blog.models import Comment


app = Celery(broker='amqp://')


@app.task
def spam_filter(comment_id, remote_addr=None):
    logger = spam_filter.get_logger()
    logger.info('Running spam filter for comment %s', comment_id)

    comment = Comment.objects.get(pk=comment_id)
    current_domain = Site.objects.get_current().domain
    akismet = Akismet(settings.AKISMET_KEY, 'http://{0}'.format(current_domain))
    if not akismet.verify_key():
        raise ImproperlyConfigured('Invalid AKISMET_KEY')


    is_spam = akismet.comment_check(user_ip=remote_addr,
                        comment_content=comment.comment,
                        comment_author=comment.name,
                        comment_author_email=comment.email_address)
    if is_spam:
        comment.is_spam = True
        comment.save()

    return is_spam
```
