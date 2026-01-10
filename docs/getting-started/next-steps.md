---
subtitle: Next Steps
description: Celery 后续步骤指南 - 详细演示如何在项目中使用 Celery，包括任务调用、工作进程管理、Canvas 工作流程设计、路由配置和远程控制功能。
---

# 后续步骤

[快速上手](./first-steps-with-celery.md) 指南故意保持简洁。在本指南中，我将更详细地演示 Celery 提供的功能，包括如何为您的应用程序和库添加 Celery 支持。

本文档并未记录 Celery 的所有功能和最佳实践，因此建议您也阅读：[用户指南](../user-guide/application.md)。

## 在项目里使用 Celery

### 项目 {#project-layout}

```console title="目录结构"
src
└── proj
    ├── __init__.py
    ├── celery.py
    └── tasks.py
```

```python title="proj/celery.py" linenums="1"
from celery import Celery

app = Celery(
    'proj',
    broker='redis://localhost:6379/0',
    backend='redis://localhost:6379/0',
    include=['proj.tasks']
)

app.conf.update(
    result_expires=3600,
)


if __name__ == '__main__':
    app.start()
```

在此模块中，您创建了我们的 `Celery` 实例（也称为 *app*）。要在项目中使用 Celery，您只需导入此实例。

- `broker` 参数指定要使用的代理 URL。
- `backend` 参数指定要使用的结果后端。它用于跟踪任务状态和结果。您可能希望为应用程序使用不同的后端。它们都有不同的优缺点。如果不需要结果，最好禁用它们。也可以通过设置 `@task(ignore_result=True)` 选项为单个任务禁用结果。
- `include` 参数是工作进程启动时要导入的模块列表。您需要在此处添加我们的任务模块，以便工作进程能够找到我们的任务。

```python title="proj/tasks.py" linenums="1"
from .celery import app


@app.task
def add(x, y):
    return x + y


@app.task
def mul(x, y):
    return x * y


@app.task
def xsum(numbers):
    return sum(numbers)
```

### 启动 Worker 进程

可以使用 `celery` 程序启动工作进程（根据示例项目布局，您需要在 `proj` 目录的上一级目录中运行工作进程，该目录是 `src`）：

```console
$ celery -A proj worker -l INFO
celery@MBP.local v5.6.0 (recovery)

macOS-15.7.3-arm64-arm-64bit-Mach-O 2025-12-28 11:43:42

[config]
.> app:         proj:0x1081096a0
.> transport:   redis://localhost:6379/0
.> results:     redis://localhost:6379/0
.> concurrency: 10 (prefork)
.> task events: OFF (enable -E to monitor tasks in this worker)

[queues]
.> celery           exchange=celery(direct) key=celery


[tasks]
  . proj.tasks.add
  . proj.tasks.mul
  . proj.tasks.xsum

[2025-12-28 11:43:42,407: INFO/MainProcess] Connected to redis://localhost:6379/0
[2025-12-28 11:43:42,410: INFO/MainProcess] mingle: searching for neighbors
[2025-12-28 11:43:43,429: INFO/MainProcess] mingle: all alone
[2025-12-28 11:43:43,463: INFO/MainProcess] celery@MBP.local ready.
```

可以通过传递 `--help` 标志来获取命令行参数的完整列表：

```console
$ celery worker --help
Usage: celery worker [OPTIONS]

  Start worker instance.

  Examples
  --------

  $ celery --app=proj worker -l INFO
  $ celery -A proj worker -l INFO -Q hipri,lopri
  $ celery -A proj worker --concurrency=4
  $ celery -A proj worker --concurrency=1000 -P eventlet
  $ celery worker --autoscale=10,0

Worker Options:
  -n, --hostname HOSTNAME         Set custom hostname (e.g., 'w1@%%h').
                                  Expands: %%h (hostname), %%n (name) and %%d,
                                  (domain).
  -D, --detach                    Start worker as a background process.
  -S, --statedb PATH              Path to the state database. The extension
                                  '.db' may be appended to the filename.
  -l, --loglevel [DEBUG|INFO|WARNING|ERROR|CRITICAL|FATAL]
                                  Logging level.
  -O, --optimization [default|fair]
                                  Apply optimization profile.
  --prefetch-multiplier <prefetch multiplier>
                                  Set custom prefetch multiplier value for
                                  this worker instance.
  --disable-prefetch              Disable broker prefetching. The worker will
                                  only fetch a task when a process slot is
                                  available. Only supported with Redis
                                  brokers.

Pool Options:
  -c, --concurrency <concurrency>
                                  Number of child processes processing the
                                  queue.  The default is the number of CPUs
                                  available on your system.
  -P, --pool [prefork|eventlet|gevent|solo|processes|threads|custom]
                                  Pool implementation.
  -E, --task-events, --events     Send task-related events that can be
                                  captured by monitors like celery events,
                                  celerymon, and others.
  --time-limit FLOAT              Enables a hard time limit (in seconds
                                  int/float) for tasks.
  --soft-time-limit FLOAT         Enables a soft time limit (in seconds
                                  int/float) for tasks.
  --max-tasks-per-child INTEGER   Maximum number of tasks a pool worker can
                                  execute before it's terminated and replaced
                                  by a new worker.
  --max-memory-per-child INTEGER  Maximum amount of resident memory, in KiB,
                                  that may be consumed by a child process
                                  before it will be replaced by a new one.  If
                                  a single task causes a child process to
                                  exceed this limit, the task will be
                                  completed and the child process will be
                                  replaced afterwards. Default: no limit.

Queue Options:
  --purge, --discard
  -Q, --queues COMMA SEPARATED LIST
  -X, --exclude-queues COMMA SEPARATED LIST
  -I, --include COMMA SEPARATED LIST

Features:
  --without-gossip
  --without-mingle
  --without-heartbeat
  --heartbeat-interval INTEGER
  --autoscale <MIN WORKERS>, <MAX WORKERS>

Embedded Beat Options:
  -B, --beat
  -s, --schedule-filename, --schedule TEXT
  --scheduler TEXT

Daemonization Options:
  -f, --logfile TEXT  Log destination; defaults to stderr
  --pidfile TEXT      PID file path; defaults to no PID file
  --uid TEXT          Drops privileges to this user ID
  --gid TEXT          Drops privileges to this group ID
  --umask TEXT        Create files and directories with this umask
  --executable TEXT   Override path to the Python executable

Options:
  --help  Show this message and exit.
```

这些选项在：[工作进程](../user-guide/workers.md) 中有更详细的描述。

### 停止 Worker 进程

要停止工作进程，只需按 ++ctrl+c++。工作进程支持的信号列表在：[工作进程](../user-guide/workers.md) 中有详细说明。

### 在后台运行

在生产环境中，在 **后台运行** Worker 进程，这在：[守护进程化](../user-guide/daemonizing.md) 中有详细描述。

守护进程化脚本使用 `celery multi` 命令在后台启动一个或多个工作进程：

```console
$ celery multi start w1 -A proj -l INFO
celery multi v5.6.0 (recovery)
> Starting nodes...
	> w1@MBP.local: OK
```

也可以 **重新启动**：

```console
$ celery multi restart w1 -A proj -l INFO
celery multi v5.6.0 (recovery)
> Stopping nodes...
	> w1@MBP.local: TERM -> 87734
> Waiting for 1 node -> 87734.....
	> w1@MBP.local: OK
> Restarting node w1@MBP.local: OK
> Waiting for 1 node -> None...
```

或者 **停止**：

```console
$ celery multi stop w1 -A proj -l INFO
celery multi v5.6.0 (recovery)
> Stopping nodes...
	> w1@MBP.local: TERM -> 87879
```

`stop` 命令是异步的，因此它不会等待工作进程关闭。您可能希望使用 `stopwait` 命令代替，它确保所有当前正在执行的任务在退出前都已完成：

```console
$ celery multi stopwait w1 -A proj -l INFO
celery multi v5.6.0 (recovery)
> Stopping nodes...
	> w1@MBP.local: TERM -> 88126
> Waiting for 1 node -> 88126.....
	> w1@MBP.local: OK
> w1@MBP.local: DOWN
> Waiting for 1 node -> None...
```

!!! note "`celery multi` 不存储有关工作进程的信息，因此在重新启动时需要使用相同的命令行参数。只有相同的 pidfile 和 logfile 参数在停止时必须使用。"

默认情况下，它会在当前目录中创建 pid 和日志文件。为防止多个工作进程相互覆盖启动，建议将这些文件放在专用目录中：

```console
mkdir -p /var/run/celery
mkdir -p /var/log/celery
celery multi start w1 -A proj -l INFO --pidfile=/var/run/celery/%n.pid --logfile=/var/log/celery/%n%I.log
```

使用 multi 命令，可以启动多个工作进程，并且有一个强大的命令行语法来为不同的工作进程指定参数，例如：

```console
celery multi start 10 -A proj -l INFO -Q:1-3 images,video -Q:4,5 data -Q default -L:4,5 debug
```

更多示例请参阅 API 参考中的 `celery.bin.multi` 模块。

关于 `--app` 参数

`--app` 参数指定要使用的 Celery 应用实例，格式为 `module.path:attribute`

但它也支持快捷形式。如果只指定了包名，它会尝试按以下顺序搜索应用实例：

使用 `--app=proj`：

1. 名为 `proj.app` 的属性，或
2. 名为 `proj.celery` 的属性，或
3. 模块 `proj` 中值为 Celery 应用程序的任何属性，或 如果以上都未找到，它会尝试名为 `proj.celery` 的子模块：

4. 名为 `proj.celery.app` 的属性，或
5. 名为 `proj.celery.celery` 的属性，或
6. 模块 `proj.celery` 中值为 Celery 应用程序的任何属性。

此方案模仿了文档中使用的实践 -- 即，对于单个包含模块使用 `proj:app`，对于较大的项目使用 `proj.celery:app`。

## 调用任务

您可以使用 `delay()` 方法调用任务：

```pycon
>>> from proj.tasks import add
>>> add.delay(2, 2)
<AsyncResult: 9c939bba-5cb7-4853-bf0c-9b2483164172>
```

此方法实际上是另一个名为 `apply_async()` 的方法的星号参数快捷方式：

```pycon
>>> add.apply_async((2, 2))
<AsyncResult: 9c939bba-5cb7-4853-bf0c-9b2483164172>
```

后者使您能够指定执行选项，如运行时间（倒计时）、应发送到的队列等：

```pycon
>>> add.apply_async((2, 2), queue='lopri', countdown=10)
```

在上面的示例中，任务将被发送到名为 `lopri` 的队列（queue），并且任务最早将在消息发送后 10 秒执行。

直接应用任务将在当前进程中执行任务，因此不会发送消息：

```pycon
>>> add(2, 2)
4
```

这三种方法 - `delay()`、`apply_async()` 和应用（`__call__()`），构成了 Celery 调用 API，也用于签名。

调用 API 的更详细概述可以在：[调用指南](../user-guide/calling.md) 中找到。

每个任务调用都会被赋予一个唯一标识符（UUID）-- 这就是任务 ID。

`delay()` 和 `apply_async()` 方法返回一个 `AsyncResult` 实例，可用于跟踪任务的执行状态。但为此，您需要启用 [结果后端](./backends-and-brokers/index.md)，以便状态可以存储在某个地方。

默认情况下结果被禁用，因为没有适合每个应用程序的结果后端；要选择一个，您需要考虑每个单独后端的缺点。对于许多任务，保留返回值甚至不是很有用，因此这是一个明智的默认设置。另请注意，结果后端不用于监控任务和工作进程：为此，Celery 使用专用的事件消息（请参阅 [监控指南](../user-guide/monitoring.md)）。

如果您配置了结果后端，可以检索任务的返回值：

```pycon
>>> res = add.delay(2, 2)
>>> res.get(timeout=1)
4
```

您可以通过查看 `id` 属性找到任务的 ID：

```pycon
>>> res.id
d6b3aea2-fb9b-4ebc-8da4-848818db9114
```

如果任务引发异常，您还可以检查异常和回溯，实际上 `result.get()` 默认会传播任何错误：

```pycon
>>> res = add.delay(2, '2')
>>> res.get(timeout=1)
```

```pytb
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "celery/result.py", line 221, in get
    return self.backend.wait_for_pending(
  File "celery/backends/asynchronous.py", line 195, in wait_for_pending
    return result.maybe_throw(callback=callback, propagate=propagate)
  File "celery/result.py", line 333, in maybe_throw
    self.throw(value, self._to_remote_traceback(tb))
  File "celery/result.py", line 326, in throw
    self.on_ready.throw(*args, **kwargs)
  File "vine/promises.py", line 244, in throw
    reraise(type(exc), exc, tb)
  File "vine/five.py", line 195, in reraise
    raise value
TypeError: unsupported operand type(s) for +: 'int' and 'str'
```

如果您不希望错误传播，可以通过传递 `propagate` 来禁用它：

```pycon
>>> res.get(propagate=False)
TypeError("unsupported operand type(s) for +: 'int' and 'str'")
```

在这种情况下，它将返回引发的异常实例 -- 因此要检查任务是成功还是失败，您必须使用结果实例上的相应方法：

```pycon
>>> res.failed()
True

>>> res.successful()
False
```

那么它如何知道任务（task）是否失败了呢？它可以通过查看任务的 *state* 来找出：

```pycon
>>> res.state
'FAILURE'
```

任务（task）只能处于单一状态，但它可以经历多个状态。典型任务的阶段可以是：

```text
PENDING -> STARTED -> SUCCESS
```

started 状态是一个特殊状态，只有在启用 `task_track_started` 设置或为任务设置 `@task(track_started=True)` 选项时才会记录。

pending 状态实际上不是记录的状态，而是任何未知任务 ID 的默认状态：您可以从以下示例中看到这一点：

```pycon
>>> from proj.celery import app

>>> res = app.AsyncResult('this-id-does-not-exist')
>>> res.state
'PENDING'
```

如果任务被重试，阶段可能会变得更加复杂。为了演示，对于一个重试两次的任务，阶段将是：

```text
PENDING -> STARTED -> RETRY -> STARTED -> RETRY -> STARTED -> SUCCESS
```

要了解更多关于任务状态的信息，您应该查看任务用户指南中的 [任务状态](../user-guide/tasks.md#task-states) 部分。

调用任务在：[调用指南](../user-guide/calling.md) 中有详细描述。

## Canvas：设计工作流程

您刚刚学习了如何使用任务的 `delay` 方法调用任务，这通常就是您所需要的全部。但有时您可能希望将任务调用的签名传递给另一个进程或作为另一个函数的参数，为此 Celery 使用称为 *签名（signatures）* 的东西。

签名将单个任务调用的参数和执行选项包装起来，以便可以传递给函数，甚至可以序列化并通过网络发送。

您可以使用参数 `(2, 2)` 和 10 秒的倒计时来为 `add` 任务创建签名，如下所示：

```pycon
>>> add.signature((2, 2), countdown=10)
tasks.add(2, 2)
```

还有一个使用星号参数的快捷方式：

```pycon
>>> add.s(2, 2)
tasks.add(2, 2)
```

### 又是那个调用 API

签名实例也支持调用 API，这意味着它们有 `delay()` 和 `apply_async()` 方法。

但有一个区别，签名可能已经指定了参数签名。`add` 任务接受两个参数，因此指定两个参数的签名将构成一个完整的签名：

```pycon
>>> s1 = add.s(2, 2)
>>> res = s1.delay()
>>> res.get()
4
```

但是，您也可以创建不完整的签名来创建我们称之为 *partials* 的东西：

```pycon
# 不完整的部分：add(?, 2)
>>> s2 = add.s(2)
```

`s2` 现在是一个部分签名，需要另一个参数才能完成，这可以在调用签名时解决：

```pycon
# 解决部分：add(8, 2)
>>> res = s2.delay(8)
>>> res.get()
10
```

在这里，您添加了参数 8，它被前置到现有参数 `2` 之前，形成了 `add(8, 2)` 的完整签名。

关键字参数也可以稍后添加；这些参数随后会与任何现有的关键字参数合并，但新参数优先：

```pycon
>>> s3 = add.s(2, 2, debug=True)
>>> s3.delay(debug=False)   # debug 现在是 False。
```

如前所述，签名支持调用 API：这意味着

- `sig.apply_async(args=(), kwargs={}, **options)` 使用可选的局部参数和局部关键字参数调用签名。还支持局部执行选项。

- `sig.delay(*args, **kwargs)` `apply_async` 的星号参数版本。任何参数都将前置到签名中的参数，关键字参数将与任何现有键合并。

所以这一切似乎非常有用，但您实际上可以用这些做什么呢？要了解这一点，我必须介绍 canvas 基本元素（primitives）...

### 基本元素

这些基本元素本身就是签名对象，因此它们可以以任意方式组合来构成复杂的工作流程。

!!! note

    这些示例检索结果，因此要尝试它们，您需要配置结果后端。上面的示例项目已经做到了这一点（请参阅 `Celery()` 的 backend 参数）。

让我们看一些例子：

#### Groups

`group` 并行调用任务列表，并返回一个特殊的结果实例，让您可以检查组的结果并按顺序检索返回值。

```pycon
>>> from celery import group
>>> from proj.tasks import add

>>> group(add.s(i, i) for i in range(10))().get()
[0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
```

- Partial group

```pycon
>>> g = group(add.s(i) for i in range(10))
>>> g(10).get()
[10, 11, 12, 13, 14, 15, 16, 17, 18, 19]
```

#### Chains

任务可以链接在一起，以便在一个任务返回后调用另一个任务：

```pycon
>>> from celery import chain
>>> from proj.tasks import add, mul

# (4 + 4) * 8
>>> chain(add.s(4, 4) | mul.s(8))().get()
64
```

- Partial chain

```pycon
>>> # (? + 4) * 8
>>> g = chain(add.s(4) | mul.s(8))
>>> g(4).get()
64
```

也可以这样写：

```pycon
>>> (add.s(4, 4) | mul.s(8))().get()
64
```

#### Chords

Chords 是一个带有回调的 Group：

```pycon
>>> from celery import chord
>>> from proj.tasks import add, xsum

>>> chord((add.s(i, i) for i in range(10)), xsum.s())().get()
90
```

Chain 到另一个任务的 Group 将自动转换为 Chord：

```pycon
>>> (group(add.s(i, i) for i in range(10)) | xsum.s())().get()
90
```

由于这些基本元素（primitives）都是签名（signature）类型，它们几乎可以以任何方式组合，例如：

```pycon
>>> upload_document.s(file) | group(apply_filter.s() for filter in filters)
```

请务必在 [画布](../user-guide/canvas.md) 用户指南中阅读更多关于工作流程的内容。
·
## 路由

Celery 支持 AMQP 提供的所有路由功能，但也支持简单的路由，其中消息被发送到命名队列。

`task_routes` 设置使您能够按名称路由任务，并将所有内容集中在一个位置：

```python
app.conf.update(
    task_routes = {
        'proj.tasks.add': {'queue': 'hipri'},
    },
)
```

您还可以在运行时使用 `apply_async()` 的 `queue` 参数指定队列：

```pycon
>>> from proj.tasks import add
>>> add.apply_async((2, 2), queue='hipri')
```

然后，您可以通过指定 `celery worker -Q` 选项使工作进程从此队列消费：

```console
celery -A proj worker -Q hipri
```

您可以使用逗号分隔的列表指定多个队列。例如，您可以使工作进程同时从默认队列和 `hipri` 队列消费，其中默认队列由于历史原因命名为 `celery`：

```console
celery -A proj worker -Q hipri,celery
```

队列的顺序无关紧要，因为工作进程将给予队列相等的权重。

要了解更多关于路由的信息，包括利用 AMQP 路由的全部功能，请参阅 [路由指南](../user-guide/routing.md)。

## 远程控制

如果您使用 RabbitMQ (AMQP)、Redis 或 Qpid 作为代理（broker），那么您可以在运行时控制和检查工作进程。

例如，您可以看到工作进程当前正在处理哪些任务：

```console
celery -A proj inspect active
```

这是通过使用广播消息实现的，因此所有远程控制命令都会被集群中的每个工作进程接收。

您还可以使用 `celery inspect --destination` 选项指定一个或多个工作进程来处理请求。这是一个逗号分隔的工作进程主机名列表：

```console
celery -A proj inspect active --destination=celery@example.com
```

如果未提供目标，则每个工作进程都会处理并回复请求。

`celery inspect` 命令包含不更改工作进程中任何内容的命令；它只返回有关工作进程内部情况的信息和统计信息。要查看检查命令列表，您可以执行：

```console
celery -A proj inspect --help
```

然后是 `celery control` 命令，它包含在运行时实际更改工作进程内容的命令：

```console
celery -A proj control --help
```

例如，您可以强制工作进程启用事件消息（用于监控任务和工作进程）：

```console
celery -A proj control enable_events
```

启用事件后，您可以启动事件转储器来查看工作进程正在做什么：

```console
celery -A proj events --dump
```

或者您可以启动 [curses](https://docs.python.org/zh-cn/3.14/howto/curses.html){target="_blank"} 界面：

```console
celery -A proj events
```

完成监控后，您可以再次禁用事件：

```console
celery -A proj control disable_events
```

:program:`celery status` 命令也使用远程控制命令，并显示集群中在线工作进程的列表：

```console
celery -A proj status
```

您可以在 [监控指南](../user-guide/monitoring.md) 中阅读更多关于 `celery` 命令和监控的信息。

## 时区

所有时间和日期，在内部和消息中都使用 UTC 时区。

当工作进程收到消息时，例如设置了倒计时，它会将该 UTC 时间转换为本地时间。如果您希望使用与系统时区不同的时区，则必须使用 `timezone` 设置进行配置：

```python
app.conf.timezone = 'Europe/London'
```

## 优化

默认配置未针对吞吐量进行优化。默认情况下，它试图在多个短任务和较少长任务之间走中间路线，这是吞吐量和公平调度之间的折衷。

如果您有严格的公平调度要求，或者希望优化吞吐量，那么您应该阅读 [优化指南](../user-guide/optimizing.md)。
