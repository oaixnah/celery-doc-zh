---
subtitle: Introduction
description: Celery是一个用Python编写的分布式任务队列系统，支持异步任务处理、高可用性、快速性能和灵活扩展。文档详细介绍任务队列概念、Celery特性、支持的代理和并发模式，以及如何快速上手使用Celery进行分布式任务处理。
---

# 介绍

## 什么是任务队列？

任务队列是一种在多个线程或机器之间分配工作的机制。

任务队列的输入称为任务的工作单元。专用的工作进程持续监控任务队列以执行新的工作。

Celery通过消息进行通信，通常使用代理在客户端和工作进程之间进行中介。要启动任务，客户端会向队列添加消息，然后代理将该消息传递给工作进程。

一个Celery系统可以包含多个工作进程和代理，从而实现高可用性和水平扩展。

Celery是用Python编写的，但该协议可以用任何语言实现。除了Python之外，还有Node.js的node-celery_、[PHP客户端](https://github.com/gjedeer/celery-php)、[gocelery](https://github.com/gocelery/gocelery)、[gopher-celery](https://github.com/marselester/gopher-celery)和[rusty-celery](https://github.com/rusty-celery/rusty-celery)。

语言互操作性也可以通过暴露HTTP端点并让任务请求它（webhooks）来实现。

## 我需要什么？

??? warning "版本要求"

    Celery 5.5.x 版本运行在：
    
    - Python ❨3.8, 3.9, 3.10, 3.11, 3.12, 3.13❩
    - PyPy3.9+ ❨v7.3.12+❩
    
    如果您运行的是较旧版本的Python，则需要运行较旧版本的Celery：
    
    - Python 3.7: Celery 5.2 或更早版本
    - Python 3.6: Celery 5.1 或更早版本
    - Python 2.7: Celery 4.x 系列
    - Python 2.6: Celery 3.1 系列或更早版本
    - Python 2.5: Celery 3.0 系列或更早版本
    - Python 2.4: Celery 2.2 系列或更早版本
    
    Celery是一个资金有限的项目，
    因此我们不支持Microsoft Windows。
    请不要打开与该平台相关的任何问题。

*Celery*需要一个消息传输器来发送和接收消息。
RabbitMQ和Redis代理传输器功能完整，
但也支持许多其他实验性解决方案，包括
使用SQLite进行本地开发。

*Celery*可以在单台机器、多台机器甚至跨数据中心运行。

## 开始使用

如果您是第一次尝试使用Celery，或者您没有跟上3.1版本的开发并且来自之前的版本，
那么您应该阅读我们的入门教程：

- [快速上手](first-steps-with-celery.md){target="_blank"}
- [后续步骤](next-steps.md){target="_blank"}

## Celery是...

### 简单

Celery易于使用和维护，并且*不需要配置文件*。

以下是您可以制作的最简单的应用程序之一：

```python
from celery import Celery

app = Celery('hello', broker='amqp://guest@localhost//')

@app.task
def hello():
    return 'hello world'
```

### 高可用性

工作进程和客户端在连接丢失或失败时会自动重试，
并且一些代理支持*主/主*或*主/副本*复制方式的高可用性。

### 快速

单个Celery进程每分钟可以处理数百万个任务，
具有亚毫秒级的往返延迟（使用RabbitMQ、
librabbitmq和优化设置）。

### 灵活

*Celery*的几乎每个部分都可以扩展或单独使用，
自定义池实现、序列化器、压缩方案、日志记录、
调度器、消费者、生产者、代理传输器等等。

## 它支持

### 代理

- [Redis](./backends-and-brokers/redis.md)
- [RabbitMQ](./backends-and-brokers/rabbitmq.md)
- [Amazon SQS](./backends-and-brokers/sqs.md)

### 并发

- prefork（多进程）
- [Eventlet](../user-guide/concurrency/eventlet.md), [gevent](../user-guide/concurrency/gevent.md)
- thread（多线程）
- `solo`（单线程）

### 结果存储

- AMQP, Redis
- Memcached
- SQLAlchemy, Django ORM
- Apache Cassandra, Elasticsearch, Riak
- MongoDB, CouchDB, Couchbase, ArangoDB
- Amazon DynamoDB, Amazon S3
- Microsoft Azure Block Blob, Microsoft Azure Cosmos DB
- Google Cloud Storage
- 文件系统

### 序列化

- *pickle*, *json*, *yaml*, *msgpack*
- *zlib*, *bzip2* 压缩
- 加密消息签名

## 特性

### 监控

工作进程会发出监控事件流，
内置和外部工具使用这些事件来实时告诉您
集群正在做什么。[了解更多](../user-guide/monitoring.md)。

### 工作流

简单和复杂的工作流可以使用我们称为"canvas"的
一组强大原语来组合，
包括分组、链式、分块等。[了解更多](../user-guide/canvas.md)。

### 时间和速率限制

您可以控制每秒/分钟/小时可以执行多少个任务，
或者任务可以运行多长时间，这可以设置为
默认值、针对特定工作进程或针对每个任务类型单独设置。[了解更多](../user-guide/workers.md#time-limits)。

### 调度

您可以以秒为单位或使用
[`datetime`](https://docs.python.org/dev/library/datetime.html#datetime.datetime){target="_blank"}指定任务运行时间，或者您可以使用
基于简单间隔的周期性任务来处理重复事件，
或者使用支持分钟、小时、星期几、月份日期和
年份月份的Crontab表达式。[了解更多](../user-guide/periodic-tasks.md#starting-the-scheduler)。

### 资源泄漏保护

[`--max-tasks-per-child`](https://docs.celeryq.dev/en/stable/reference/cli.html#cmdoption-celery-worker-max-tasks-per-child)
选项用于处理用户任务泄漏资源的情况，比如内存或
文件描述符，这些完全超出您的控制范围。[了解更多](../user-guide/workers.md#max-tasks-per-child)。

### 用户组件

每个工作进程组件都可以自定义，并且用户可以
定义其他组件。工作进程使用"bootsteps"构建——
一个依赖关系图，可以精细控制工作进程的
内部结构。

- [Eventlet](http://eventlet.net/)
- [gevent](http://gevent.org/)
