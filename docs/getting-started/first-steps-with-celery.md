---
subtitle: First Steps with Celery
description: Celery快速入门教程：从零开始学习如何使用Celery分布式任务队列，包括安装配置、创建任务、运行工作进程、调用任务、保存结果和故障排除等完整指南。
---

# 快速上手

Celery 是一个功能齐全的任务队列。它易于使用，让无需了解其解决的全部复杂性即可开始使用。它围绕最佳实践设计，使的产品能够扩展并与其他语言集成，并且配备了在生产环境中运行此类系统所需的工具和支持。

## 1. 选择代理（Broker）

Celery 需要一个发送和接收消息的解决方案；通常这以称为 *消息代理（message broker）* 的独立服务形式出现。

以 [Redis](https://redis.io/){target="_blank"} 为例。使用 Docker 快速运行：

```console
docker run -d -p 6379:6379 redis
```

## 2. 安装 Celery

=== "uv"

    ```console
    uv add celery
    ```

=== "pip"

    ```console
    pip install celery
    ```

## 3. 创建 Celery 应用程序（app）

首先需要一个 Celery 实例。称之为 *Celery 应用程序* 或简称为 *app*。由于此实例用作在 Celery 中想要执行的所有操作的入口点，例如创建任务（task）和管理工作进程（worker），因此其他模块必须能够导入它。

在本教程中，我们将所有内容包含在单个模块中，但对于较大的项目，需要创建一个 [项目](next-steps.md#project-layout)。

创建文件 `tasks.py`：

```python title="tasks.py" linenums="1"
from celery import Celery

app = Celery('tasks', broker='redis://localhost:6379/0')

@app.task
def add(x, y):
    return x + y
```

第3行：`Celery` 实例的第一个参数是**当前模块名称**。这仅在任务在 `__main__` 模块中定义时需要，以便可以自动生成名称。第二个参数是代理（broker）关键字参数，指定使用的消息代理（message broker）的 URL。这里使用 [Redis](backends-and-brokers/redis.md){target="_blank"}。

第6行：定义了一个名为 `add` 的单个任务（task）函数，它接受两个参数 `x` 和 `y`，并返回它们的和。

## 4. 运行 Celery Worker 服务

现在可以通过使用 `worker` 参数执行程序来运行 worker 进程：

```console
celery -A tasks worker --loglevel=INFO
```

要获取可用命令行选项的完整列表，请执行：

```console
celery worker --help
```

## 5. 调用任务

要调用任务（task），可以使用 `delay` 方法。它是 `apply_async()` 方法的一个便捷快捷方式，后者对任务执行提供更大的控制（请参阅：[调用任务](../user-guide/calling.md)）

```pycon
>>> from tasks import add
>>> add.delay(4, 4)
```

该任务（task）现在已被之前启动的工作（worker）进程处理。可以通过查看工作（worker）进程的控制台输出来验证这一点。

调用任务（task）后，会返回一个 `AsyncResult` 实例。这可用于检查任务的状态、等待任务完成，或获取其返回值（或者如果任务失败，获取异常和回溯信息）。

默认情况下未启用结果。为了进行远程过程调用或在数据库中跟踪任务结果，需要配置 Celery 使用结果后端。这将在下一节中描述。

## 6. 保存结果

如果想跟踪任务的状态，Celery 需要将状态存储或发送到某个地方。有几个内置的结果后端可供选择：[SQLAlchemy](https://www.sqlalchemy.org/){target="_blank"}/[Django](https://djangoproject.com/){target="_blank"} ORM、[MongoDB](https://www.mongodb.com/){target="_blank"}，或者可以定义自己的后端。

对于此示例，我们使用 `rpc` 结果后端，它将状态作为瞬态消息发送回来。后端通过 `Celery` 的 `backend` 参数指定（或者如果选择使用配置模块，则通过 `result_backend` 设置指定）。因此，可以修改 `tasks.py` 文件中的这一行来启用 `rpc://` 后端：

```python
app = Celery('tasks', backend='rpc://', broker='pyamqp://')
```

或者，如果想使用 Redis 作为结果后端（result backend），但仍使用 RabbitMQ 作为消息代理（message broker）（一种流行的组合）：

```python
app = Celery('tasks', backend='redis://localhost', broker='pyamqp://')
```

要了解有关结果后端（result backend）的更多信息，请参阅：[结果后端](./backends-and-brokers/index.md)。

现在配置了结果后端（result backend），重新启动工作（worker）进程，关闭当前的 Python 会话并重新导入 `tasks` 模块以使更改生效。这次将保留调用任务时返回的 `AsyncResult` 实例：

```pycon
>>> from tasks import add    # 关闭并重新打开以获取更新的 'app'
>>> result = add.delay(4, 4)
```

`ready()` 方法返回任务是否已完成处理：

```pycon
>>> result.ready()
False
```

可以等待结果完成，但这很少使用，因为它将异步调用转换为同步调用：

```pycon
>>> result.get(timeout=1)
8
```

如果任务引发异常，`get()` 方法将重新引发异常，但可以通过指定 `propagate` 参数来覆盖此行为：

```pycon
>>> result.get(propagate=False)
```

如果任务引发异常，还可以访问原始回溯：

```pycon
>>> result.traceback
```

!!! warning

    后端使用资源来存储和传输结果。为确保资源被释放，最终必须对调用任务后返回的每个 `AsyncResult` 实例调用 `get()` 或 `forget()` 方法。

有关完整的结果对象参考，请参阅 `celery.result`。

## 7. 配置 {#celerytut-configuration}

Celery 就像一个家用电器，不需要太多配置即可运行。它有一个输入和一个输出。输入必须连接到代理（broker），输出可以选择性地连接到结果后端（result backend）。但是，如果仔细查看背面，会发现一个盖子，里面有很多滑块、刻度盘和按钮：这就是配置。

默认配置应该足以满足大多数用例，但有许多选项可以配置，以使 Celery 完全按照需要工作。阅读可用选项是熟悉可配置内容的好方法。可以在 [配置](../user-guide/configuration.md) 参考中阅读有关选项的信息。

配置可以直接在应用程序上设置，也可以通过专用配置模块设置。
例如，可以通过更改 `task_serializer` 设置来配置用于序列化任务负载的默认序列化器：

```python
app.conf.task_serializer = 'json'
```

如果要一次配置多个设置，可以使用 `update()` 方法：

```python
app.conf.update(
    task_serializer='json',
    accept_content=['json'],  # 忽略其他内容
    result_serializer='json',
    timezone='Europe/Oslo',
    enable_utc=True,
)
```

对于较大的项目，建议使用专用配置模块。不鼓励硬编码周期性任务间隔和任务路由选项。最好将这些内容保存在集中位置。这对于库尤其重要，因为它使用户能够控制其任务的行为。集中配置还将允许的系统管理员在系统出现问题时进行简单更改。

可以通过调用 `config_from_object()` 方法告诉的 Celery 实例使用配置模块：

```python
app.config_from_object('celeryconfig')
```

此模块通常称为 `celeryconfig`，但可以使用任何模块名称。

在上述情况下，必须可以从当前目录或 Python 路径加载名为 `celeryconfig.py` 的模块。它可能看起来像这样：

```python title="celeryconfig.py"
broker_url = 'pyamqp://'
result_backend = 'rpc://'

task_serializer = 'json'
result_serializer = 'json'
accept_content = ['json']
timezone = 'Europe/Oslo'
enable_utc = True
```

要验证的配置文件是否正常工作且不包含任何语法错误，可以尝试导入它：

```console
python -m celeryconfig
```

有关配置选项的完整参考，请参阅：[配置](../user-guide/configuration.md)。

为了演示配置文件的强大功能，以下是如何将有问题的任务路由到专用队列的方法：

```python title="celeryconfig.py"
task_routes = {
    'tasks.add': 'low-priority',
}
```

或者，可以通过速率限制任务而不是路由它，以便每分钟只能处理 10 个此类任务（10/m）：

```python title="celeryconfig.py"
task_annotations = {
    'tasks.add': {'rate_limit': '10/m'},
}
```

如果使用 RabbitMQ 或 Redis 作为代理，那么还可以指示工作进程在运行时为任务设置新的速率限制：

```console
celery -A tasks control rate_limit tasks.add 10/m
worker@example.com: OK
    new rate limit set successfully
```

有关任务路由的更多信息，请参阅：[路由任务](../user-guide/routing.md)，有关注释的更多信息，请参阅 `task_annotations` 设置，或者有关远程控制命令以及如何监控工作进程的更多信息，请参阅：[监控](../user-guide/monitoring.md)。

## 8. 下一步

如果想了解更多信息，应该继续学习：[下一步](./next-steps.md) 教程，之后可以阅读：[用户指南](../user-guide/application.md)。

## 9. 故障排除

[QA](../faq.md) 中也有故障排除部分。

### 工作进程无法启动：权限错误

- 如果使用 Debian、Ubuntu 或其他基于 Debian 的发行版：

    Debian 最近将 `/dev/shm` 特殊文件重命名为 `/run/shm`。

    一个简单的解决方法是创建一个符号链接：

    ```console
    ln -s /run/shm /dev/shm
    ```

- 其他系统：

    如果提供 `celery worker --pidfile`、`celery worker --logfile` 或 `celery worker --statedb` 参数中的任何一个，则必须确保它们指向启动工作进程的用户可写和可读的文件或目录。

### 结果后端不工作或任务始终处于 `PENDING` 状态

默认情况下，所有任务都是 `PENDING`，因此该状态最好命名为 "unknown"。Celery 在发送任务时不会更新状态，任何没有历史记录的任务都被假定为挂起（毕竟知道任务 ID）。

1. 确保任务没有启用 `ignore_result`。

    启用此选项将强制工作进程跳过更新状态。

2. 确保 `task_ignore_result` 设置未启用。

3. 确保没有任何旧的工作进程仍在运行。

    很容易意外启动多个工作进程，因此请确保在启动新工作进程之前正确关闭了先前的工作进程。

    一个未配置预期结果后端的旧工作进程可能正在运行并劫持任务。

    可以将 `celery worker --pidfile` 参数设置为绝对路径以确保不会发生这种情况。

4. 确保客户端配置了正确的后端。

    如果由于某种原因，客户端配置的后端与工作进程不同，将无法接收结果。
    确保后端配置正确：

    ```pycon
    >>> result = task.delay()
    >>> print(result.backend)
    ```
