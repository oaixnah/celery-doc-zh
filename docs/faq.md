---
hide:
  - navigation
description: Celery常见问题解答 - 全面解析Python分布式任务队列的疑问，涵盖任务调度、消息代理、故障排除、安全配置、性能优化等核心问题，提供详细的解决方案和最佳实践指南。
---

# 常见问题解答

## 概述

### 我应该使用 Celery 来做什么？

**回答：** [队列化一切，让所有人满意](https://decafbad.com/blog/2008/07/04/queue-everything-and-delight-everyone) 是一篇很好的文章，描述了为什么在 Web 环境中使用队列。

以下是一些常见的用例：

- 在后台运行某些任务。例如，尽快完成 Web 请求，
  然后逐步更新用户页面。
  这给用户留下了良好性能和"响应迅速"的印象，即使
  实际工作可能需要一些时间。

- 在 Web 请求完成后运行某些任务。

- 通过异步执行和使用重试机制来确保某些任务完成。

- 调度定期工作。

在某种程度上还包括：

- 分布式计算。

- 并行执行。

## 误解

### Celery 真的包含 50,000 行代码吗？

**回答：** 不，这个以及类似的大数字已经在多个地方被报告过。

截至本文撰写时的数字是：

- 核心：7,141 行代码
- 测试：14,209 行
- 后端、贡献、兼容性工具：9,032 行

代码行数不是一个有用的指标，所以即使 Celery 确实包含 50k 行代码，你也不能从这样的数字中得出任何结论。

### Celery 有很多依赖项吗？

一个常见的批评是 Celery 使用了太多的依赖项。
这种担忧背后的理由很难想象，特别是考虑到代码重用是现代软件开发中对抗复杂性的既定方式，而且现在添加依赖项的成本非常低，因为像 pip 和 PyPI 这样的包管理器使得安装和维护依赖项的麻烦成为过去。

Celery 在过程中已经替换了几个依赖项，当前的依赖项列表是：

#### celery

- `kombu`

Kombu 是 Celery 生态系统的一部分，是用于发送和接收消息的库。它也是使我们能够支持许多不同消息代理的库。它也被 OpenStack 项目和许多其他项目使用，验证了将其与 Celery 代码库分离的选择。

- `billiard`

Billiard 是 Python 多处理模块的一个分支，包含许多性能和稳定性改进。最终目标是这些改进有一天会被合并回 Python。

它也用于与不包含多处理模块的旧版 Python 版本的兼容性。

#### kombu

Kombu 依赖于以下包：

- `amqp`

底层的纯 Python amqp 客户端实现。AMQP 作为默认代理，这是一个自然的依赖项。

### Celery 是重量级的吗？

Celery 在内存占用和性能方面都带来了非常小的开销。

但请注意，默认配置没有针对时间或空间进行优化，请参阅 [优化指南](optimizing.md) 以获取更多信息。

### Celery 依赖于 pickle 吗？

**回答：** 不，Celery 可以支持任何序列化方案。

我们内置支持 JSON、YAML、Pickle 和 msgpack。每个任务都与一个内容类型相关联，因此你甚至可以使用 pickle 发送一个任务，使用 JSON 发送另一个任务。

默认的序列化支持曾经是 pickle，但从 4.0 开始默认现在是 JSON。如果你需要发送复杂的 Python 对象作为任务参数，你可以使用 pickle 作为序列化格式。

如果你需要与其他语言通信，你应该使用适合该任务的序列化格式，这基本上意味着任何不是 pickle 的序列化器。

你可以设置全局默认序列化器、特定任务的默认序列化器，
甚至是发送单个任务实例时要使用的序列化器。

### Celery 只适用于 Django 吗？

**回答：** 不，你可以在任何框架中使用 Celery，无论是 Web 框架还是其他框架。

### 我必须使用 AMQP/RabbitMQ 吗？

**回答：** 不，虽然推荐使用 RabbitMQ，但你也可以使用 Redis、SQS 或 Qpid。

Redis 作为代理的性能不如 AMQP 代理，但 RabbitMQ 作为代理和 Redis 作为结果存储的组合是常用的。如果你有严格的可靠性要求，鼓励你使用 RabbitMQ 或其他 AMQP 代理。一些传输也使用轮询，因此它们可能会消耗更多资源。然而，如果你由于某种原因无法使用 AMQP，请随意使用这些替代方案。它们可能适用于大多数用例，并请注意上述观点并非特定于 Celery；如果之前使用 Redis/数据库作为队列对你有效，现在可能仍然有效。如果需要，你以后可以随时升级。

### Celery 是多语言的吗？

**回答：** 是的。

`celery.bin.worker` 是 Celery 在 Python 中的实现。如果某种语言有 AMQP 客户端，创建该语言的 worker 应该不会太困难。一个 Celery worker 只是一个连接到代理来处理消息的程序。

此外，还有另一种实现语言独立性的方法，那就是使用 REST 任务，不是将任务作为函数，而是将它们作为 URL。有了这个信息，你甚至可以创建简单的 Web 服务器，实现代码的预加载。只需暴露一个执行操作的端点，并创建一个任务来对该端点执行 HTTP 请求。

你也可以使用 [Flower's](https://flower.readthedocs.io) [REST API](https://flower.readthedocs.io/en/latest/api.html#post--api-task-async-apply-(.+) 来调用任务。

## 故障排除

### MySQL 抛出死锁错误，我该怎么办？

**回答：** MySQL 默认的隔离级别设置为 `REPEATABLE-READ`（可重复读），如果您并不真正需要这个级别，请将其设置为 `READ-COMMITTED`（读已提交）。您可以通过在 :file:`my.cnf` 文件中添加以下内容来实现：：

    [mysqld]
    transaction-isolation = READ-COMMITTED

有关 InnoDB 事务模型的更多信息，请参阅 [MySQL - InnoDB 事务模型和锁定](https://dev.mysql.com/doc/refman/5.1/en/innodb-transaction-model.html)。

### Worker 什么都不做，只是挂起

**回答：** 请参阅 `MySQL 抛出死锁错误，我该怎么办？`，或 `为什么 Task.delay/apply\*/worker 只是挂起？`。

### 任务结果不可靠地返回

**回答：** 如果您使用数据库后端来存储结果，特别是使用 MySQL，请参阅 `MySQL 抛出死锁错误，我该怎么办？`。

### 为什么 Task.delay/apply\*/worker 只是挂起？

**回答：** 某些 AMQP 客户端存在一个错误，如果无法验证当前用户、密码不匹配或用户没有访问指定虚拟主机的权限，就会导致挂起。请务必检查您的代理日志（对于 RabbitMQ，在大多数系统上是 :file:`/var/log/rabbitmq/rabbit.log`），通常其中会包含描述原因的消息。

### 它能在 FreeBSD 上工作吗？

**回答：** 这取决于具体情况；

当使用 RabbitMQ（AMQP）和 Redis 传输时，它应该可以开箱即用。

对于其他传输方式，会使用兼容性 prefork 池，这需要可用的 POSIX 信号量实现，自 FreeBSD 8.x 以来，FreeBSD 默认启用了此功能。对于较旧版本的 FreeBSD，您必须在内核中启用 POSIX 信号量并手动重新编译 billiard。

幸运的是，Viktor Petersson 编写了一个教程，帮助您在 FreeBSD 上开始使用 Celery：
http://www.playingwithwire.com/2009/10/how-to-get-celeryd-to-work-on-freebsd/

### 我遇到了 `IntegrityError: Duplicate Key` 错误。为什么？

**回答：** 请参阅 `MySQL 抛出死锁错误，我该怎么办？`。

### 为什么我的任务没有被处理？

**回答：** 使用 RabbitMQ 时，您可以通过运行以下命令查看当前有多少消费者正在接收任务：

```console
rabbitmqctl list_queues -p <myvhost> name messages consumers
Listing queues ...
celery     2891    2
```

这显示任务队列中有 2891 条消息等待处理，有两个消费者正在处理它们。

队列永远不会被清空的一个原因可能是您有一个陈旧的 worker 进程劫持了消息。如果 worker 没有正确关闭，就可能发生这种情况。

当 worker 接收到消息时，代理会等待消息被确认后才将其标记为已处理。在消费者正确关闭之前，代理不会将该消息重新发送给另一个消费者。

如果您遇到此问题，必须手动杀死所有 worker 并重新启动它们：

```console
pkill 'celery worker'

# - 如果您没有 pkill，请使用：
# ps auxww | awk '/celery worker/ {print $2}' | xargs kill
```

您可能需要等待一段时间，直到所有 worker 完成执行任务。 如果长时间后仍然挂起，您可以使用强制方式杀死它们：

```console
pkill -9 'celery worker'

# - 如果您没有 pkill，请使用：
# ps auxww | awk '/celery worker/ {print $2}' | xargs kill -9
```

### 为什么我的任务不运行？

**回答：** 可能存在语法错误阻止了任务模块的导入。

您可以通过手动执行任务来检查 Celery 是否能够运行该任务：

```pycon
>>> from myapp.tasks import MyPeriodicTask
>>> MyPeriodicTask.delay()
```

查看 worker 的日志文件，看看它是否能够找到任务，或者是否发生了其他错误。

### 为什么我的周期性任务不运行？

**回答：** 请参阅 `为什么我的任务不运行？`。

### 如何清除所有等待的任务？

**回答：** 您可以使用 `celery purge` 命令清除所有配置的任务队列：

```console
celery -A proj purge
```

或者通过编程方式：

```pycon
>>> from proj.celery import app
>>> app.control.purge()
1753
```

如果您只想清除特定队列中的消息，
必须使用 AMQP API 或 `celery amqp` 实用程序：

```console
celery -A proj amqp queue.purge <queue name>
```

数字 1753 是已删除消息的数量。

您还可以在启动 worker 时启用 `celery worker --purge` 选项，以便在 worker 启动时清除消息。

### 我已经清除了消息，但队列中仍有消息？

**回答：** 任务在实际执行后才会被确认（从队列中移除）。在 worker 接收到任务后，需要一些时间才能实际执行，特别是如果已经有大量任务等待执行时。未被确认的消息会被 worker 保留，直到它关闭与代理（AMQP 服务器）的连接。当该连接关闭时（例如，因为 worker 被停止），代理会将任务重新发送给下一个可用的 worker（或者在 worker 重新启动后发送给同一个 worker），因此要正确清除队列中的等待任务，您必须停止所有 worker，然后使用 `celery.control.purge` 清除任务。

## 结果

### 如果我有指向任务的ID，如何获取任务的结果？

**回答**：使用 `task.AsyncResult`：

```pycon
>>> result = my_task.AsyncResult(task_id)
>>> result.get()
```

这将为您提供一个 `celery.result.AsyncResult` 实例使用任务的当前结果后端。

如果您需要指定自定义结果后端，或者想要使用当前应用程序的默认后端，您可以使用 `AsyncResult`：

```pycon
>>> result = app.AsyncResult(task_id)
>>> result.get()
```

## 安全

### 使用 `pickle` 是否存在安全风险？

**回答**：确实如此，自 Celery 4.0 起，默认的序列化器已改为 JSON，以确保人们能够有意识地选择序列化器并意识到这一风险。

保护您的代理、数据库和其他传输 pickle 数据的服务免受未经授权的访问至关重要。

请注意，这不仅仅是 Celery 需要注意的问题，例如 Django 也使用 pickle 作为其缓存客户端。

对于任务消息，您可以将 `task_serializer` 设置改为 "json" 或 "yaml" 而不是 pickle。同样地，对于任务结果，您可以设置 `result_serializer`。

有关使用的格式以及在检查任务使用什么格式时的查找顺序的更多详细信息。

### 消息可以加密吗？

**回答**：一些 AMQP 代理支持使用 SSL（包括 RabbitMQ）。您可以使用 `broker_use_ssl` 设置来启用此功能。

### 以 root 身份运行 :program:`celery worker` 安全吗？

**回答**：不安全！

我们目前没有发现任何安全问题，但假设不存在安全问题是极其天真的，因此建议以非特权用户身份运行 Celery 服务（`celery worker`、`celery beat`、`celeryev` 等）。

## 消息代理

### 为什么 RabbitMQ 会崩溃？

**回答：** 如果 RabbitMQ 内存耗尽，它就会崩溃。这将在 RabbitMQ 的未来版本中得到修复。请参考 RabbitMQ 常见问题解答：
https://www.rabbitmq.com/faq.html#node-runs-out-of-memory

!!! note

    这种情况已经不再存在，RabbitMQ 2.0 及以上版本包含了一个新的持久化器，能够容忍内存不足错误。建议为 Celery 使用 RabbitMQ 2.1 或更高版本。

    如果您仍在运行旧版本的 RabbitMQ 并遇到崩溃问题，请升级！

Celery 的错误配置最终可能导致旧版本 RabbitMQ 的崩溃。即使没有崩溃，这仍然会消耗大量资源，因此了解常见的陷阱非常重要。

- 事件。

    使用 `celery worker -E` 选项运行 `celery.bin.worker` 将发送工作器内部发生事件的消息。
    
    只有在有活动监视器消费这些事件时，或者定期清除事件队列时，才应启用事件。

- AMQP 后端结果。

    当使用 AMQP 结果后端运行时，每个任务结果都将作为消息发送。如果您不收集这些结果，它们会累积起来，RabbitMQ 最终会耗尽内存。
    
    这个结果后端现在已被弃用，因此您不应该使用它。对于 rpc 风格的调用使用 RPC 后端，或者如果您需要多消费者访问结果，请使用持久化后端。
    
    默认情况下，结果会在 1 天后过期。通过配置 `result_expires` 设置来降低此值可能是个好主意。
    
    如果您不使用任务的结果，请确保设置 `ignore_result` 选项：
    
    ```python
    @app.task(ignore_result=True)
    def mytask():
        pass

    class MyTask(Task):
        ignore_result = True
    ```

### 我可以在 Celery 中使用 ActiveMQ/STOMP 吗？

**回答：** 不可以。它曾经由 `Carrot`（我们的旧消息库）支持但目前在 `Kombu`（我们的新消息库）中不受支持。

### 不使用 AMQP 代理时，哪些功能不受支持？

这是使用虚拟传输时不可用的功能不完整列表：

- 远程控制命令（仅 Redis 支持）。
- 使用事件进行监视可能无法在所有虚拟传输中工作。
- `header` 和 `fanout` 交换类型（`fanout` 由 Redis 支持）。

## 任务

### 如何在调用任务时重用相同的连接？

**回答**：请参阅 :setting:`broker_pool_limit` 设置。连接池自版本 2.5 起默认启用。

### 在 `subprocess` 中的 `sudo` 返回 `None`

有一个 `sudo` 配置选项使得没有 tty 的进程运行 `sudo` 是非法的：

```text
Defaults requiretty
```

如果在你的 `/etc/sudoers` 文件中有此配置，那么当 worker 作为守护进程运行时，任务将无法调用 `sudo`。如果你想启用此功能，则需要从 `/etc/sudoers` 中删除该行。

参考：http://timelordz.com/wiki/Apache_Sudo_Commands

### 为什么 worker 会从队列中删除无法处理的任务？

**回答**：

worker 会拒绝未知任务、编码错误的消息以及不包含适当字段的消息（根据任务消息协议）。

如果不拒绝它们，它们可能会被重复传递，导致循环。

RabbitMQ 的最新版本具有为交换配置死信队列的能力，因此被拒绝的消息会被移动到那里。

### 我可以通过名称调用任务吗？

**回答**：是的，使用 `send_task`。

你也可以使用 AMQP 客户端从任何语言通过名称调用任务：

```pycon
>>> app.send_task('tasks.add', args=[2, 2], kwargs={})
<AsyncResult: 373550e8-b9a0-4666-bc61-ace01fa4f91d>
```

要使用 `chain`、`chord` 或 `group` 与按名称调用的任务，请使用 `Celery.signature` 方法：

```pycon
>>> chain(
...     app.signature('tasks.add', args=[2, 2], kwargs={}),
...     app.signature('tasks.add', args=[1, 1], kwargs={})
... ).apply_async()
<AsyncResult: e9d52312-c161-46f0-9013-2713e6df812d>
```

### 我可以获取当前任务的 task id 吗？

**回答**：是的，当前 id 和更多信息在任务请求中可用：：

```python
@app.task(bind=True)
def mytask(self):
    cache.set(self.request.id, "Running")
```

如果你没有对任务实例的引用，可以使用 `app.current_task`：

```pycon
>>> app.current_task.request.id
```

但请注意，这可能是任何任务，无论是 worker 执行的任务，还是该任务直接调用的任务，或者是急切调用的任务。

要特别获取当前正在处理的任务，请使用 `app.current_worker_task`：

```pycon
>>> app.current_worker_task.request.id
```

!!! note

    `app.current_task` 和 `app.current_worker_task` 都可能为 `None`。

### 我可以指定自定义的 task_id 吗？

**回答**：是的，使用 `Task.apply_async` 的 `task_id` 参数：

```pycon
>>> task.apply_async(args, kwargs, task_id='…')
```

### 我可以在任务中使用装饰器吗？

**回答**：是的。

### 我可以使用自然的任务 id 吗？

**回答**：是的，但要确保它是唯一的，因为两个具有相同 id 的任务的行为是未定义的。

世界可能不会爆炸，但它们绝对可以覆盖彼此的结果。

### 我可以在另一个任务完成后运行一个任务吗？

**回答**：是的，你可以在任务内部安全地启动另一个任务。

一个常见的模式是为任务添加回调：

```python
from celery.utils.log import get_task_logger

logger = get_task_logger(__name__)

@app.task
def add(x, y):
    return x + y

@app.task(ignore_result=True)
def log_result(result):
    logger.info("log_result got: %r", result)
```

调用：

```pycon
>>> (add.s(2, 2) | log_result.s()).delay()
```

### 我可以取消任务的执行吗？

**回答**：是的，使用 `result.revoke()`：

```pycon
>>> result = add.apply_async(args=[2, 2], countdown=120)
>>> result.revoke()
```

或者如果你只有任务 id：

```pycon
>>> from proj.celery import app
>>> app.control.revoke(task_id)
```

后者还支持传递任务 id 列表作为参数。

### 为什么我的远程控制命令没有被所有 worker 接收？

**回答**：为了接收广播远程控制命令，每个 worker 节点都会基于 worker 的节点名创建一个唯一的队列名称。

如果你有多个具有相同主机名的 worker，控制命令将在它们之间以轮询方式接收。

要解决这个问题，你可以使用 `celery worker -n` 参数为每个 worker 显式设置节点名：

```console
celery -A proj worker -n worker1@%h
celery -A proj worker -n worker2@%h
```

其中 `%h` 扩展为当前主机名。

### 我可以将某些任务仅发送到某些服务器吗？

**回答**：是的，你可以使用不同的消息路由拓扑将任务路由到一个或多个 worker，并且一个 worker 实例可以绑定到多个队列。

更多信息请参阅 [路由指南](user-guide/routing.md)。

### 我可以禁用任务的预取吗？

**回答**：也许可以！AMQP 术语 "prefetch" 令人困惑，因为它仅用于描述任务预取*限制*。实际上并没有涉及真正的预取。

禁用预取限制是可能的，但这意味着 worker 将尽可能快地消耗尽可能多的任务。

你可以使用 `celery worker --disable-prefetch` 标志（或将 `worker_disable_prefetch` 设置为 `True`），以便 worker 仅在其某个进程空闲时才获取任务。此功能目前仅在使用 Redis 作为代理时受支持。

### 我可以在运行时更改周期性任务的间隔吗？

**回答**：是的，你可以使用 Django 数据库调度器，或者你可以创建一个新的调度子类并重写 `celery.schedules.schedule.is_due`：

```python
from celery.schedules import schedule

class my_schedule(schedule):

    def is_due(self, last_run_at):
        return run_now, next_time_to_check
```

### Celery 支持任务优先级吗？

**回答**：是的，RabbitMQ 自版本 3.5.0 起支持优先级，而 Redis 传输模拟了优先级支持。

你也可以通过将高优先级任务路由到不同的 worker 来优先处理工作。在现实世界中，这通常比每条消息的优先级效果更好。你可以将此与速率限制和每条消息优先级结合使用，以实现响应式系统。

### 我应该使用 retry 还是 acks_late？{#acks_late-vs-retry}

**回答**：取决于情况。不一定是二选一，你可能想要同时使用两者。

`Task.retry` 用于重试任务，特别是对于可通过 `try` 块捕获的预期错误。AMQP 事务不用于这些错误：**如果任务引发异常，它仍然会被确认！**

当 worker（由于某种原因）在执行过程中崩溃时，如果需要再次执行任务，将使用 `acks_late` 设置。重要的是要注意，worker 不会已知崩溃，如果确实崩溃，通常是一个需要人工干预的不可恢复错误（worker 或任务代码中的错误）。

在理想世界中，你可以安全地重试任何失败的任务，但这种情况很少见。想象以下任务：

```python
@app.task
def process_upload(filename, tmpfile):
    # 增加存储在数据库中的文件计数
    increment_file_counter()
    add_file_metadata_to_db(filename, tmpfile)
    copy_file_to_destination(filename, tmpfile)
```

如果这在将文件复制到目的地的过程中崩溃，世界将包含不完整的状态。当然，这不是一个关键场景，但你可能会想象更糟糕的情况。因此，为了编程的简便性，我们降低了可靠性；这是一个好的默认设置，需要此功能并知道自己在做什么的用户仍然可以启用 acks_late（并且将来希望使用手动确认）。

此外，`Task.retry` 具有 AMQP 事务不可用的功能：重试之间的延迟、最大重试次数等。

因此，对于 Python 错误使用 retry，如果你的任务是幂等的，并且需要这种可靠性级别，则结合使用 `acks_late`。

### 我可以安排任务在特定时间执行吗？

**回答**：是的。你可以使用 `Task.apply_async` 的 `eta` 参数。请注意，不建议使用遥远的 `eta` 时间。

### 我可以安全地关闭 worker 吗？

**回答**：是的，使用 `TERM` 信号。

这将告诉 worker 完成所有当前正在执行的作业并尽快关闭。只要关闭完成，即使是实验性传输也不应丢失任何任务。

你永远不应该使用 `KILL` 信号（`kill -9`）停止 `celery.bin.worker`，除非你已经尝试了几次 `TERM` 并等待了几分钟让它有机会关闭。

还要确保只杀死主 worker 进程，而不是它的任何子进程。如果你知道进程当前正在执行 worker 关闭所依赖的任务，你可以将 kill 信号定向到特定的子进程，但这意味着将为任务设置 `WorkerLostError` 状态，因此任务不会再次运行。

如果你安装了 `setproctitle` 模块，识别进程类型会更容易：

```console
pip install setproctitle
```

安装此库后，你将能够在 `ps` 列表中看到进程类型，但必须重新启动 worker 才能生效。

### 我可以在 [platform] 的后台运行 worker 吗？

**回答**：是的。

## Django

### `django-celery-beat` 创建的数据库表有什么用途？

当使用基于数据库的调度时，周期性任务的调度信息取自 `PeriodicTask` 模型，还有几个其他辅助表（`IntervalSchedule`、`CrontabSchedule`、`PeriodicTasks`）。

### `django-celery-results` 创建的数据库表有什么用途？

Django 数据库结果后端扩展需要
两个额外的模型：`TaskResult` 和 `GroupResult`。

## Windows

### Celery 是否支持 Windows？

**回答**：不支持。

自 Celery 4.x 起，由于资源不足，不再支持 Windows。

但它可能仍然有效，我们很乐意接受补丁。
