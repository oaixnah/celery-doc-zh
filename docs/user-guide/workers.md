---
subtitle: Worker
description: Celery工作进程完整指南：启动、停止、重启、并发控制、远程管理、任务撤销、时间限制、速率限制、自动扩展等详细配置与操作说明。
---

# 工作进程

## 启动 worker

!!! tip inline end "守护进程化"

    你可能希望使用守护进程化工具来在后台启动工作者。有关使用流行的服务管理器将工作者作为守护进程启动的帮助，请参阅 [守护进程指南](daemonizing.md){target="_blank"}。

你可以通过执行以下命令在前台启动工作者：

```console
celery -A proj worker -l INFO
```

有关可用命令行选项的完整列表，请参阅 `celery.bin.worker`，或者简单地执行：

```console
celery worker --help
```

你可以在同一台机器上启动多个工作者，但
请确保通过使用 `celery worker --hostname` 参数指定节点名称来命名每个单独的工作者：

```console
celery -A proj worker --loglevel=INFO --concurrency=10 -n worker1@%h
celery -A proj worker --loglevel=INFO --concurrency=10 -n worker2@%h
celery -A proj worker --loglevel=INFO --concurrency=10 -n worker3@%h
```

`hostname` 参数可以展开以下变量：

- `%h`: 主机名，包括域名。
- `%n`: 仅主机名。
- `%d`: 仅域名。

如果当前主机名是 *george.example.com*，这些变量将展开为：

| 变量   | 模板           | 结果                           |
|------|--------------|------------------------------|
| `%h` | `worker1@%h` | *worker1@george.example.com* |
| `%n` | `worker1@%n` | *worker1@george*             |
| `%d` | `worker1@%d` | *worker1@example.com*        |

## 停止 worker 进程 {#worker-stopping}

关闭应该使用 `TERM` 信号来完成。

当关闭启动时，工作进程将在实际终止之前完成所有当前正在执行的任务。如果这些任务很重要，您应该等待它完成，然后再采取任何激进的措施，比如发送 `KILL` 信号。

如果工作进程在合理的时间内无法关闭，例如因为陷入无限循环或类似情况，您可以使用 `KILL` 信号来强制终止工作进程：但请注意，当前正在执行的任务将会丢失（即，除非任务设置了 `acks_late` 选项）。

此外，由于进程无法覆盖 `KILL` 信号，工作进程将无法回收其子进程；请确保手动执行此操作。以下命令通常可以解决问题：

```console
pkill -9 -f 'celery worker'
```

如果您的系统没有 `pkill` 命令，可以使用稍长一些的版本：

```console
ps auxww | awk '/celery worker/ {print $2}' | xargs kill -9
```

### 工作进程关闭

我们将使用术语*暖关闭、软关闭、冷关闭、硬关闭*来描述工作进程关闭的不同阶段。当工作进程接收到 `TERM` 或 `QUIT` 信号时，它将启动关闭过程。`INT` (Ctrl-C) 信号在关闭过程中也会被处理，并且总是触发关闭过程的下一阶段。

#### 暖关闭（Warm Shutdown）

当工作进程接收到 `TERM` 信号时，它将启动暖关闭。工作进程将在实际终止之前完成所有当前正在执行的任务。工作进程第一次接收到 `INT` (Ctrl-C) 信号时，也会启动暖关闭。

暖关闭将停止调用 `celery.worker.worker.WorkController.start()` 并调用 `celery.worker.worker.WorkController.stop()`。

- 在暖关闭过程中，额外的 `TERM` 信号将被忽略。
- 下一个 `INT` 信号将触发关闭过程的下一阶段。

#### 冷关闭（Cold Shutdown） {#worker-cold-shutdown}

当工作进程接收到 `QUIT` 信号时，将启动冷关闭。工作进程将停止所有当前正在执行的任务并立即终止。

!!! note

    如果环境变量 `REMAP_SIGTERM` 设置为 `SIGQUIT`，工作进程在接收到 `TERM` 信号时也会启动冷关闭，而不是暖关闭。

冷关闭将停止调用 `celery.worker.worker.WorkController.start()` 并调用 `celery.worker.worker.WorkController.terminate()`。

如果暖关闭已经启动，向冷关闭的过渡将运行一个信号处理程序 `on_cold_shutdown` 来取消主进程中所有当前正在执行的任务，并可能触发 软关闭。

#### 软关闭（Soft Shutdown） {#worker-soft-shutdown}

软关闭是一种有时间限制的暖关闭，在冷关闭之前启动。工作进程将允许 `worker_soft_shutdown_timeout` 秒的时间让所有当前正在执行的任务完成，然后终止。如果达到时间限制，工作进程将启动冷关闭并取消所有当前正在执行的任务。如果在软关闭期间接收到 `QUIT` 信号，工作进程将取消所有当前正在执行的任务，但仍会等待时间限制结束才终止，这给了工作进程一个更优雅地执行冷关闭的机会。

软关闭默认是禁用的，以保持与 冷关闭 行为的向后兼容性。要启用软关闭，请将 `worker_soft_shutdown_timeout` 设置为一个正浮点数值。如果没有任务在运行，软关闭将被跳过。要强制软关闭，*同时*启用 `worker_enable_soft_shutdown_on_idle` 设置。

!!! warning

    如果工作进程没有运行任何任务但保留了 ETA 任务，除非启用了 `worker_enable_soft_shutdown_on_idle` 设置，否则不会启动软关闭，这可能导致在冷关闭期间丢失任务。在使用 ETA 任务时，建议启用空闲时的软关闭。尝试不同的 :setting:`worker_soft_shutdown_timeout` 值，找到最适合您设置的配置，以将任务丢失的风险降到最低。

例如，当设置 `worker_soft_shutdown_timeout=3` 时，工作进程将允许 3 秒的时间让所有当前正在执行的任务完成，然后终止。如果达到时间限制，工作进程将启动冷关闭并取消所有当前正在执行的任务。

```console
[INFO/MainProcess] Task myapp.long_running_task[6f748357-b2c7-456a-95de-f05c00504042] received
[WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 1/2000s
[WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 2/2000s
[WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 3/2000s
^C
worker: Hitting Ctrl+C again will initiate cold shutdown, terminating all running tasks!

worker: Warm shutdown (MainProcess)
[WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 4/2000s
[WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 5/2000s
[WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 6/2000s
^C
worker: Hitting Ctrl+C again will terminate all running tasks!
[WARNING/MainProcess] Initiating Soft Shutdown, terminating in 3 seconds
[WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 7/2000s
[WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 8/2000s
[WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 9/2000s
[WARNING/MainProcess] Restoring 1 unacknowledged message(s)
```

- 下一个 `QUIT` 信号将取消软关闭中仍在运行的任务，但工作进程仍会等待时间限制结束才终止。
- 下一个（第 2 个）`QUIT` 或 `INT` 信号将触发关闭过程的下一阶段。

#### 硬关闭（Hard Shutdown）

硬关闭主要用于本地或调试目的，允许连续发送 `INT` (Ctrl-C) 信号来强制工作进程立即终止。工作进程将停止所有当前正在执行的任务，并通过在主进程中引发 `WorkerTerminate` 异常来立即终止。

例如，注意下面日志中的 `^C`（使用 `INT` 信号在不同阶段之间移动）：

```console
[INFO/MainProcess] Task myapp.long_running_task[7235ac16-543d-4fd5-a9e1-2d2bb8ab630a] received
[WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 1/2000s
[WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 2/2000s
^C
worker: Hitting Ctrl+C again will initiate cold shutdown, terminating all running tasks!

worker: Warm shutdown (MainProcess)
[WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 3/2000s
[WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 4/2000s
^C
worker: Hitting Ctrl+C again will terminate all running tasks!
[WARNING/MainProcess] Initiating Soft Shutdown, terminating in 10 seconds
[WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 5/2000s
[WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 6/2000s
^C
Waiting gracefully for cold shutdown to complete...

worker: Cold shutdown (MainProcess)
^C[WARNING/MainProcess] Restoring 1 unacknowledged message(s)
```

!!! warning

    日志 `Restoring 1 unacknowledged message(s)` 具有误导性，因为在硬关闭后不能保证消息会被恢复。暖关闭 允许在暖关闭和冷关闭之间添加一个时间窗口，从而提高关闭过程的优雅性。

## 重启 worker 进程

要重启工作进程，您应该发送 `TERM` 信号并启动一个新的实例。在开发环境中管理工作进程最简单的方法是使用 `celery multi`：

```console
celery multi start 1 -A proj -l INFO -c4 --pidfile=/var/run/celery/%n.pid
celery multi restart 1 --pidfile=/var/run/celery/%n.pid
```

对于生产部署，您应该使用初始化脚本或进程监控系统（参见 [守护进程指南](daemonizing.md){target="_blank"}）。

除了停止然后启动工作进程来重启之外，您也可以使用 `HUP` 信号来重启工作进程。请注意，工作进程将负责自行重启，因此这种方法容易出现问题，不建议在生产环境中使用：

```console
kill -HUP $pid
```

!!! note

    通过 `HUP` 重启仅在工作进程作为守护进程在后台运行时有效（它没有控制终端）。

    `HUP` 在 macOS 上被禁用，因为该平台存在限制。

## 连接丢失时自动重连到代理

除非 `broker_connection_retry_on_startup` 设置为 False，Celery 将在第一次连接丢失后自动重试重新连接到代理。`broker_connection_retry` 控制是否自动重试重新连接到代理以进行后续重连。

如果 `worker_cancel_long_running_tasks_on_connection_loss` 设置为 True，Celery 还将取消当前正在运行的任何长时间运行的任务。

由于消息代理不跟踪在连接丢失之前已经获取了多少任务，Celery 将通过当前正在运行的任务数量乘以 `worker_prefetch_multiplier` 来减少预取计数。预取计数将在每次连接丢失前正在运行的任务完成后逐渐恢复到允许的最大值。

此功能默认启用，但可以通过将 False 设置为 `worker_enable_prefetch_count_reduction` 来禁用。

## 进程信号

工作进程的主进程重写了以下信号：

| Signal | Action                        |
|--------|-------------------------------|
| `TERM` | 温和关闭，等待任务完成。                  |
| `QUIT` | 强制关闭，尽快终止                     |
| `USR1` | 转储所有活动线程的堆栈跟踪。                |
| `USR2` | 远程调试，参见 `celery.contrib.rdb`。 |

## 文件路径中的变量

`--logfile <celery worker --logfile>`、`--pidfile <celery worker --pidfile>` 和 `--statedb <celery worker --statedb>` 的文件路径参数可以包含变量，工作进程会展开这些变量：

### 节点名称替换

- `%p`: 完整节点名称
- `%h`: 主机名，包括域名
- `%n`: 仅主机名
- `%d`: 仅域名
- `%i`: 预分叉池进程索引，如果是主进程则为 0
- `%I`: 带分隔符的预分叉池进程索引

例如，如果当前主机名是 `george@foo.example.com`，那么这些变量会展开为：

- `--logfile=%p.log` -> `george@foo.example.com.log`
- `--logfile=%h.log` -> `foo.example.com.log`
- `--logfile=%n.log` -> `george.log`
- `--logfile=%d.log` -> `example.com.log`

### 预分叉池进程索引

预分叉池进程索引说明符会根据最终需要打开文件的进程展开为不同的文件名。

这可以用于为每个子进程指定一个日志文件。

请注意，即使进程退出或使用了自动缩放/`maxtasksperchild`/时间限制，
数字也会保持在进程限制范围内。也就是说，这个数字是*进程索引*，
而不是进程计数或进程ID。

* `%i` - 池进程索引，如果是主进程则为 0

    其中 `-n worker1@example.com -c2 -f %n-%i.log` 会产生三个日志文件：

    - `worker1-0.log` (主进程)
    - `worker1-1.log` (池进程 1)
    - `worker1-2.log` (池进程 2)

* `%I` - 带分隔符的池进程索引

    其中 `-n worker1@example.com -c2 -f %n%I.log` 会产生三个日志文件：

    - `worker1.log` (主进程)
    - `worker1-1.log` (池进程 1)
    - `worker1-2.log` (池进程 2)

## 并发性 {#worker-concurrency}

默认情况下使用多进程来执行任务的并发执行，但你也可以使用 [Eventlet](concurrency/eventlet.md){target="_blank"}。工作进程/线程的数量可以使用 `celery worker --concurrency` 参数进行更改，默认值为机器上可用的CPU数量。

!!! info "进程数量（多进程/prefork池）"

    更多的池进程通常更好，但存在一个临界点，超过这个点添加更多的池进程会对性能产生负面影响。甚至有证据表明，运行多个工作实例可能比单个工作实例性能更好。例如，3个工作实例，每个有10个池进程。你需要进行实验来找到最适合你的数字，因为这取决于应用程序、工作负载、任务运行时间和其他因素。

## 远程控制

!!! tip inline end "`celery` 命令"

    `celery` 程序用于从命令行执行远程控制命令。它支持下面列出的所有命令。

| 支持     | 说明                                                          |
|--------|-------------------------------------------------------------|
| pool   | *prefork, eventlet, gevent, thread*, blocking:*solo* (参见注释) |
| broker | *amqp, redis*                                               |

工作器具有使用高优先级广播消息队列进行远程控制的能力。命令可以定向到所有工作器，或特定的工作器列表。

命令也可以有回复。客户端可以等待并收集这些回复。由于没有中央机构知道集群中有多少工作器可用，也无法估计有多少工作器可能发送回复，因此客户端有一个可配置的超时时间——回复到达的截止时间（以秒为单位）。此超时时间默认为一秒。如果工作器在截止时间内没有回复，并不一定意味着工作器没有回复，或者更糟的是已经死亡，而可能只是由网络延迟或工作器处理命令速度较慢引起的，因此请相应地调整超时时间。

除了超时时间，客户端还可以指定等待的最大回复数。如果指定了目标，此限制将设置为目标主机的数量。

!!! note

    `solo` 池支持远程控制命令，但任何正在执行的任务都会阻塞任何等待的控制命令，因此如果工作器非常繁忙，其用途有限。在那种情况下，您必须增加客户端等待回复的超时时间。

### `broadcast` 函数

这是用于向工作器发送命令的客户端函数。一些远程控制命令也有使用 `broadcast()` 在后台的更高级接口，例如 `rate_limit()` 和 `ping()`。

发送 `rate_limit()` 命令和关键字参数：

```pycon
>>> app.control.broadcast('rate_limit', arguments={'task_name': 'myapp.mytask', 'rate_limit': '200/m'})
```

这将异步发送命令，无需等待回复。要请求回复，您必须使用 `reply` 参数：

```pycon
>>> app.control.broadcast('rate_limit', {'task_name': 'myapp.mytask', 'rate_limit': '200/m'}, reply=True)
[{'worker1.example.com': 'New rate limit set successfully'},
 {'worker2.example.com': 'New rate limit set successfully'},
 {'worker3.example.com': 'New rate limit set successfully'}]
```

使用 `destination` 参数，您可以指定要接收命令的工作器列表：

```pycon
>>> app.control.broadcast('rate_limit', {
...     'task_name': 'myapp.mytask',
...     'rate_limit': '200/m'}, reply=True,
...                             destination=['worker1@example.com'])
[{'worker1.example.com': 'New rate limit set successfully'}]
```

当然，使用更高级的接口来设置速率限制要方便得多，但有些命令只能通过 `broadcast()` 请求。

## 命令

### `revoke`：撤销任务

| 支持      | 说明                                                            |
|---------|---------------------------------------------------------------|
| pool    | all, terminate only supported by prefork, eventlet and gevent |
| broker  | *amqp, redis*                                                 |
| command | `celery -A proj control revoke <task_id>`                     |

所有工作节点都会在内存中保留已撤销任务ID的记录，可以是内存中或持久化到磁盘。

!!! note

    内存中保留的已撤销任务的最大数量可以通过 `CELERY_WORKER_REVOKES_MAX` 环境变量指定，默认值为50000。当超过限制时，撤销将在10800秒（3小时）后过期。可以使用 `CELERY_WORKER_REVOKE_EXPIRES` 环境变量更改此值。

    成功任务的内存限制也可以通过 `CELERY_WORKER_SUCCESSFUL_MAX` 和 `CELERY_WORKER_SUCCESSFUL_EXPIRES` 环境变量设置，默认值分别为1000和10800。

当工作节点收到撤销请求时，它将跳过执行该任务，但不会终止已经在执行的任务，除非设置了 `terminate` 选项。

!!! note

    terminate选项是管理员在任务卡住时的最后手段。它不是用于终止任务，而是用于终止执行任务的进程，并且该进程可能在发送信号时已经开始处理另一个任务，因此您绝不能以编程方式调用此功能。

如果设置了 `terminate`，处理任务的子进程将被终止。默认发送的信号是 `TERM`，但您可以使用 `signal` 参数指定。信号可以是Python标准库中 `signal` 模块定义的任何信号的大写名称。

终止任务也会撤销它。

**示例**

```pycon
>>> result.revoke()

>>> AsyncResult(id).revoke()

>>> app.control.revoke('d9078da5-9915-40a0-bfa1-392c7bde42ed')

>>> app.control.revoke('d9078da5-9915-40a0-bfa1-392c7bde42ed',
...                    terminate=True)

>>> app.control.revoke('d9078da5-9915-40a0-bfa1-392c7bde42ed',
...                    terminate=True, signal='SIGKILL')
```

### 撤销多个任务

revoke方法也接受列表参数，可以一次性撤销多个任务。

**示例**

```pycon
>>> app.control.revoke([
...    '7993b0aa-1f0b-4780-9af0-c47c0858b3f2',
...    'f565793e-b041-4b2b-9ca4-dca22762a55d',
...    'd9d35e03-2997-42d0-a13e-64a66b88a618',
])
```

自版本3.1起，`GroupResult.revoke` 方法就利用了此功能。

### 持久化撤销

撤销任务的工作原理是向所有工作节点发送广播消息，工作节点随后在内存中保留已撤销任务的列表。当工作节点启动时，它将与集群中的其他工作节点同步已撤销的任务。

已撤销任务的列表存储在内存中，因此如果所有工作节点都重启，已撤销ID的列表也会消失。如果要在重启之间保留此列表，需要使用 `--statedb` 参数指定一个文件来存储这些信息：

```console
celery -A proj worker -l INFO --statedb=/var/run/celery/worker.state
```

或者如果使用 `celery multi`，您需要为每个工作节点实例创建一个文件，因此使用 `%n` 格式来扩展当前节点名称：

```console
celery multi start 2 -l INFO --statedb=/var/run/celery/%n.state
```

请注意，撤销功能需要远程控制命令正常工作。目前仅RabbitMQ（amqp）和Redis支持远程控制命令。

### `revoke_by_stamped_headers`：通过标记头撤销任务

| 支持      | 说明                                                            |
|---------|---------------------------------------------------------------|
| pool    | all, terminate only supported by prefork and eventlet         |
| broker  | *amqp, redis*                                                 |
| command | `celery -A proj control revoke_by_stamped_headers <header=value>` |

此命令类似于 `revoke()`，但不是指定任务ID，而是将标记头指定为键值对，每个具有与键值对匹配的标记头的任务都将被撤销。

!!! warning

    已撤销头的映射在重启之间不是持久的，因此如果重启工作节点，已撤销的头将丢失，需要重新映射。

!!! warning

    如果工作池并发性高且启用了terminate，此命令的性能可能较差，因为它必须迭代所有正在运行的任务来查找具有指定标记头的任务。

**示例**

```pycon
>>> app.control.revoke_by_stamped_headers({'header': 'value'})

>>> app.control.revoke_by_stamped_headers({'header': 'value'}, terminate=True)

>>> app.control.revoke_by_stamped_headers({'header': 'value'}, terminate=True, signal='SIGKILL')
```

### 通过标记头撤销多个任务

`revoke_by_stamped_headers` 方法也接受列表参数，可以通过多个头或多个值进行撤销。

**示例**

```pycon
>> app.control.revoke_by_stamped_headers({
...    'header_A': 'value_1',
...    'header_B': ['value_2', 'value_3'],
})
```

这将撤销所有具有标记头 `header_A` 且值为 `value_1` 的任务，以及所有具有标记头 `header_B` 且值为 `value_2` 或 `value_3` 的任务。

**CLI 示例**

```console
celery -A proj control revoke_by_stamped_headers stamped_header_key_A=stamped_header_value_1 stamped_header_key_B=stamped_header_value_2

celery -A proj control revoke_by_stamped_headers stamped_header_key_A=stamped_header_value_1 stamped_header_key_B=stamped_header_value_2 --terminate

celery -A proj control revoke_by_stamped_headers stamped_header_key_A=stamped_header_value_1 stamped_header_key_B=stamped_header_value_2 --terminate --signal=SIGKILL
```

## 时间限制 {#time-limits}

| 支持      | 说明                                                            |
|---------|---------------------------------------------------------------|
| pool    | *prefork/gevent (参见下面的注释)*                              |

!!! question inline end "软限制还是硬限制？"

    时间限制设置为两个值：`soft`（软限制）和`hard`（硬限制）。软时间限制允许任务捕获异常，在终止前进行清理：硬超时无法捕获，会强制终止任务。

单个任务可能永远运行，如果您有很多任务在等待永远不会发生的事件，您将无限期地阻止工作者处理新任务。防御这种情况发生的最佳方法是启用时间限制。

时间限制（`--time-limit`）是任务在执行的进程被终止并由新进程替换之前可以运行的最大秒数。您还可以启用软时间限制（`--soft-time-limit`），这会在硬时间限制终止任务之前引发一个任务可以捕获的异常以进行清理：

```python
from myapp import app
from celery.exceptions import SoftTimeLimitExceeded

@app.task
def mytask():
    try:
        do_work()
    except SoftTimeLimitExceeded:
        clean_up_in_a_hurry()
```

时间限制也可以使用 `task_time_limit` / `task_soft_time_limit` 设置来配置。您还可以使用 `AsyncResult.get()` 函数的 `timeout` 参数为客户端操作指定时间限制。

!!! note

    时间限制目前在不支持 `SIGUSR1` 信号的平台上不起作用。

!!! note

    gevent 池不实现软时间限制。此外，如果任务正在阻塞，它不会强制执行硬时间限制。

### 运行时更改时间限制

| 支持      | 说明                                                            |
|---------|---------------------------------------------------------------|
| broker  | *amqp, redis*                                                 |

有一个远程控制命令可以让您更改任务的软硬时间限制——名为 `time_limit`。

将 `tasks.crawl_the_web` 任务的时间限制更改为软限制一分钟，硬限制两分钟的示例：

```pycon
>>> app.control.time_limit('tasks.crawl_the_web', soft=60, hard=120, reply=True)
[{'worker1.example.com': {'ok': 'time limits set successfully'}}]
```

只有时间限制更改后开始执行的任务才会受到影响。

## 速率限制

### 运行时更改速率限制

示例：将 `myapp.mytask` 任务的速率限制更改为每分钟最多执行200个该类型的任务：

```pycon
>>> app.control.rate_limit('myapp.mytask', '200/m')
```

上面的示例没有指定目标，因此更改请求将影响集群中的所有工作实例。如果您只想影响特定的工作器列表，可以包含 `destination` 参数：

```pycon
>>> app.control.rate_limit('myapp.mytask', '200/m', destination=['celery@worker1.example.com'])
```

!!! warning

    这不会影响启用了 `worker_disable_rate_limits` 设置的工作器。

## 每个子进程最大任务数设置 {#max-tasks-per-child}

| 支持   | 说明        |
|------|-----------|
| pool | *prefork* |

使用此选项，您可以配置工作进程在被新进程替换之前可以执行的最大任务数。

这在您有无法控制的内存泄漏时非常有用，例如来自闭源C扩展的内存泄漏。

该选项可以通过工作进程的 `celery worker --max-tasks-per-child` 参数或使用 `worker_max_tasks_per_child` 设置来配置。

## 每个子进程的最大内存设置

| 支持   | 说明        |
|------|-----------|
| pool | *prefork* |

通过此选项，您可以配置工作进程在被新进程替换之前可以执行的最大驻留内存量。

如果您有无法控制的内存泄漏（例如来自闭源C扩展），这将非常有用。

该选项可以使用工作进程的 `celery worker --max-memory-per-child` 参数或使用 `worker_max_memory_per_child` 设置来配置。

## 自动扩展

| 支持   | 说明                  |
|------|---------------------|
| pool | *prefork*, *gevent* |

*autoscaler*（自动扩展器）组件用于根据负载动态调整池的大小：

- 当有工作需要处理时，自动扩展器会添加更多的池进程；当工作负载较低时，开始移除进程。

通过 `celery worker --autoscale` 选项启用自动扩展功能，该选项需要两个数字：池进程的最大和最小数量：

```text
--autoscale=AUTOSCALE
     Enable autoscaling by providing
     max_concurrency,min_concurrency.  Example:
       --autoscale=10,3 (always keep 3 processes, but grow to
      10 if necessary).
```

您也可以通过继承 `celery.worker.autoscale.Autoscaler` 类来为自动扩展器定义自己的规则。一些指标的想法包括平均负载或可用内存量。您可以使用 `worker_autoscaler` 设置来指定自定义的自动扩展器。

## 队列

一个工作实例可以从任意数量的队列中消费消息。默认情况下，它将消费在 `task_queues` 设置中定义的所有队列（如果未指定，则回退到名为 `celery` 的默认队列）。

您可以在启动时通过向 `celery worker -Q` 选项提供逗号分隔的队列列表来指定要从哪些队列消费：

```console
celery -A proj worker -l INFO -Q foo,bar,baz
```

如果队列名称在 `task_queues` 中定义，它将使用该配置，但如果未在队列列表中定义，Celery 将自动为您生成一个新队列（取决于 `task_create_missing_queues` 选项）。

您还可以使用远程控制命令 `add_consumer` 和 `cancel_consumer` 在运行时告诉工作器开始和停止从队列消费。

### 添加消费者

`add_consumer` 控制命令将告诉一个或多个工作器开始从队列消费消息。此操作是幂等的。

要告诉集群中的所有工作器开始从名为 "`foo`" 的队列消费，您可以使用 `celery control` 程序：

```console
celery -A proj control add_consumer foo
-> worker1.local: OK
    started consuming from u'foo'
```

如果您想指定特定的工作器，可以使用 `celery control --destination` 参数：

```console
celery -A proj control add_consumer foo -d celery@worker1.local
```

同样可以使用 `add_consumer()` 方法动态完成：

```pycon
>>> app.control.add_consumer('foo', reply=True)
[{u'worker1.local': {u'ok': u"already consuming from u'foo'"}}]

>>> app.control.add_consumer('foo', reply=True,
...                          destination=['worker1@example.com'])
[{u'worker1.local': {u'ok': u"already consuming from u'foo'"}}]
```

到目前为止，我们只展示了使用自动队列的示例，如果您需要更多控制，还可以指定交换器、路由键甚至其他选项：

```pycon
>>> app.control.add_consumer(
...     queue='baz',
...     exchange='ex',
...     exchange_type='topic',
...     routing_key='media.*',
...     options={
...         'queue_durable': False,
...         'exchange_durable': False,
...     },
...     reply=True,
...     destination=['w1@example.com', 'w2@example.com'])
```

### 取消消费者

您可以使用 `cancel_consumer` 控制命令通过队列名称取消消费者。

要强制集群中的所有工作器取消从队列消费，您可以使用 `celery control` 程序：

```console
celery -A proj control cancel_consumer foo
```

`celery control --destination` 参数可用于指定一个工作器或工作器列表来执行该命令：

```console
celery -A proj control cancel_consumer foo -d celery@worker1.local
```

您也可以使用 `cancel_consumer()` 方法以编程方式取消消费者：

```pycon
>>> app.control.cancel_consumer('foo', reply=True)
[{u'worker1.local': {u'ok': u"no longer consuming from u'foo'"}}]
```

### 活动队列列表

您可以使用 `active_queues` 控制命令获取工作器正在消费的队列列表：

```console
celery -A proj inspect active_queues
[...]
```

与所有其他远程控制命令一样，这也支持 `celery inspect --destination` 参数，用于指定应回复请求的工作器：

```console
$ celery -A proj inspect active_queues -d celery@worker1.local
[...]
```

这也可以通过使用 `active_queues()` 方法以编程方式完成：

```pycon
>>> app.control.inspect().active_queues()
[...]

>>> app.control.inspect(['worker1.local']).active_queues()
[...]
```

## 检查工作进程

`inspect` 允许您检查正在运行的工作进程。它在底层使用远程控制命令。

您也可以使用 `celery` 命令来检查工作进程，它支持与 `control` 接口相同的命令。

```pycon
>>> # 检查所有节点。
>>> i = app.control.inspect()

>>> # 指定多个节点进行检查。
>>> i = app.control.inspect(['worker1.example.com',
                            'worker2.example.com'])

>>> # 指定单个节点进行检查。
>>> i = app.control.inspect('worker1.example.com')
```

### 已注册任务的转储

您可以使用 `registered()` 获取工作进程中已注册任务的列表：

```pycon
>>> i.registered()
[{'worker1.example.com': ['tasks.add', 'tasks.sleeptask']}]
```

### 当前执行任务的转储

您可以使用 `active()` 获取活动任务的列表：

```pycon
>>> i.active()
[{'worker1.example.com':
    [{'name': 'tasks.sleeptask',
      'id': '32666e9b-809c-41fa-8e93-5ae0c80afbbf',
      'args': '(8,)',
      'kwargs': '{}'}]}]
```

### 已调度（ETA）任务的转储

您可以使用 `scheduled()` 获取等待调度的任务列表：

```pycon
>>> i.scheduled()
[{'worker1.example.com':
    [{'eta': '2010-06-07 09:07:52', 'priority': 0,
      'request': {
        'name': 'tasks.sleeptask',
        'id': '1a7980ea-8b19-413e-91d2-0b74f3844c4d',
        'args': '[1]',
        'kwargs': '{}'}},
     {'eta': '2010-06-07 09:07:53', 'priority': 0,
      'request': {
        'name': 'tasks.sleeptask',
        'id': '49661b9a-aa22-4120-94b7-9ee8031d219d',
        'args': '[2]',
         'kwargs': '{}'}}]}]
```

!!! note

    这些是具有 ETA/倒计时参数的任务，不是周期性任务。

### 保留任务的转储

保留任务是已接收但仍在等待执行的任务。

您可以使用 `reserved()` 获取这些任务的列表：

```pycon
>>> i.reserved()
[{'worker1.example.com':
    [{'name': 'tasks.sleeptask',
      'id': '32666e9b-809c-41fa-8e93-5ae0c80afbbf',
      'args': '(8,)',
      'kwargs': '{}'}]}]
```

### 统计信息

远程控制命令 `inspect stats`（或 `stats()`）将为您提供有关工作进程的有用（或不太有用）统计信息的长列表：

```console
celery -A proj inspect stats
```

有关输出详细信息，请查阅 `stats()` 的参考文档。

## 附加命令

### 远程关机

此命令将优雅地远程关闭工作进程：

```pycon
>>> app.control.broadcast('shutdown') # 关闭所有工作进程
>>> app.control.broadcast('shutdown', destination='worker1@example.com')
```

### Ping

此命令向存活的工作进程请求 ping。工作进程回复字符串 'pong'，仅此而已。除非您指定自定义超时时间，否则它将使用默认的一秒超时时间：

```pycon
>>> app.control.ping(timeout=0.5)
[{'worker1.example.com': 'pong'},
 {'worker2.example.com': 'pong'},
 {'worker3.example.com': 'pong'}]
```

`ping()` 也支持 `destination` 参数，因此您可以指定要 ping 的工作进程：

```pycon
>>> ping(['worker2.example.com', 'worker3.example.com'])
[{'worker2.example.com': 'pong'},
 {'worker3.example.com': 'pong'}]
```

### 启用/禁用事件

您可以使用 `enable_events`、`disable_events` 命令来启用/禁用事件。
这对于使用 `celery events`/`celerymon` 临时监控工作进程很有用。

```pycon
>>> app.control.enable_events()
>>> app.control.disable_events()
```

## 编写你自己的远程控制命令

有两种类型的远程控制命令：

- 检查命令（Inspect command）

    没有副作用，通常只是返回在worker中找到的一些值，比如当前注册的任务列表、活动任务列表等。

- 控制命令（Control command）

    执行副作用操作，比如添加一个新的队列来消费。

远程控制命令在控制面板中注册，它们接受一个参数：当前的 `celery.worker.control.ControlDispatch`实例。从这里你可以访问活动的 `celery.worker.consumer.Consumer`（如果需要的话）。

这是一个增加任务预取计数的控制命令示例：

```python
from celery.worker.control import control_command

@control_command(
    args=[('n', int)],
    signature='[N=1]',  # <- 用于命令行帮助。
)
def increase_prefetch_count(state, n=1):
    state.consumer.qos.increment_eventually(n)
    return {'ok': 'prefetch count incremented'}
```

确保你将此代码添加到worker导入的模块中：这可以是定义Celery应用程序的同一个模块，或者你可以将该模块添加到 `imports` 设置中。

重启worker以便控制命令被注册，现在你可以使用 `celery control` 实用程序调用你的命令：

```console
celery -A proj control increase_prefetch_count 3
```

你也可以向:program:`celery inspect`程序添加操作，例如一个读取当前预取计数的操作：

```python
from celery.worker.control import inspect_command

@inspect_command()
def current_prefetch_count(state):
    return {'prefetch_count': state.consumer.qos.value}
```

重启worker后，你现在可以使用 `celery inspect` 程序查询这个值：

```console
celery -A proj inspect current_prefetch_count
```
