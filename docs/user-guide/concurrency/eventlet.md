---
subtitle: Eventlet
description: Celery Eventlet并发支持指南：了解如何在Celery中使用Eventlet作为替代执行池实现，实现高度可扩展的非阻塞I/O并发处理，包括启用方法、性能优势、适用场景以及与prefork池的对比。
---

# 使用 Eventlet 实现并发

## 介绍

[Eventlet](http://eventlet.net) 官网将其描述为一个用于 Python 的并发网络库，它允许您改变代码的运行方式，而不是编写方式。

- 它使用 [`epoll(4)`](http://linux.die.net/man/4/epoll) 或 [`libevent`](https://libevent.org) 来实现 `高度可扩展的非阻塞 I/O`_。
- [`协程`](https://en.wikipedia.org/wiki/Coroutine) 确保开发者使用类似于线程的阻塞式编程风格，但提供了非阻塞 I/O 的优势。
- 事件分发是隐式的：这意味着您可以轻松地从 Python 解释器使用 Eventlet，或者作为大型应用程序的一小部分使用。

Celery 支持 Eventlet 作为替代的执行池实现，在某些情况下优于 prefork。但是，您需要确保一个任务不会阻塞事件循环太长时间。通常，CPU 密集型操作与 Eventlet 配合不佳。还要注意，某些库（通常带有 C 扩展）无法进行猴子补丁，因此无法从使用 Eventlet 中受益。如果您不确定，请参考它们的文档。例如，pylibmc 不允许与 Eventlet 协作，但 psycopg2 可以（当它们都是带有 C 扩展的库时）。

prefork 池可以利用多个进程，但数量通常限制为每个 CPU 几个进程。使用 Eventlet，您可以高效地生成数百或数千个绿色线程。在一个非正式的测试中，使用 feed hub 系统，Eventlet 池每秒可以获取和处理数百个 feed，而 prefork 池处理 100 个 feed 需要 14 秒。请注意，这是异步 I/O 特别擅长的应用之一（异步 HTTP 请求）。您可能希望混合使用 Eventlet 和 prefork worker，并根据兼容性或最佳效果来路由任务。

## 启用 Eventlet

您可以使用 `celery worker -P` worker 选项来启用 Eventlet 池。

```console
celery -A proj worker -P eventlet -c 1000
```

## 示例

请参阅 Celery 发行版中的 [Eventlet 示例](https://github.com/celery/celery/tree/main/examples/eventlet) 目录，了解一些使用 Eventlet 支持的示例。
