---
subtitle: Backends and Brokers
description: Celery后端和代理全面指南：详细介绍RabbitMQ、Redis、Amazon SQS、Zookeeper、Kafka、GC PubSub等消息代理的对比分析，包括稳定性、监控支持和远程控制功能。涵盖Redis作为后端和代理的双重角色，RabbitMQ的大消息处理能力，以及SQLAlchemy与多种SQL数据库的集成方案。
---

# 后端和代理

Celery 支持多种消息传输替代方案。

## 代理概览

| 名称         | 状态           | 监控  | 远程控制 |
|------------|--------------|-----|------|
| RabbitMQ   | Stable       | Yes | Yes  |
| Redis      | Stable       | Yes | Yes  |
| Amazon SQS | Stable       | No  | No   |
| Zookeeper  | Experimental | No  | No   |
| Kafka      | Experimental | No  | No   |
| GC PubSub  | Experimental | Yes | Yes  |

实验性代理可能功能正常，但它们没有专门的维护者。

缺少监控支持意味着传输不实现事件，因此 Flower、`celery events`、`celerymon` 和其他基于事件的监控工具将无法工作。

远程控制意味着能够在运行时使用 `celery inspect` 和 `celery control` 命令（以及使用远程控制 API 的其他工具）检查和管理工作器。

## 摘要

!!! warning "本节并非对后端和代理的全面介绍。"

Celery 能够与许多不同的后端（结果存储）和代理（消息传输）进行通信和存储。

### Redis

Redis 既可以作为后端（backend）也可以作为代理（broker）。

**作为代理：** Redis 适用于快速传输小消息。大消息可能会阻塞系统。

**作为后端：** Redis 是一个超快的键值存储，使其在获取任务调用结果时非常高效。与 Redis 的设计一样，您需要考虑可用于存储数据的有限内存，以及如何处理数据持久性。如果结果持久性很重要，请考虑为后端使用另一个数据库。

### RabbitMQ

RabbitMQ 是一个代理（broker）。

**作为代理：** RabbitMQ 处理大消息比 Redis 更好，但是如果消息非常快速地大量涌入，扩展性可能成为问题，除非 RabbitMQ 运行在非常大的规模上，否则应考虑使用 Redis 或 SQS。

**作为后端：** RabbitMQ 可以通过 `rpc://` 后端存储结果。此后端为每个客户端创建单独的临时队列。

*注意：RabbitMQ（作为代理）和 Redis（作为后端）经常一起使用。如果结果存储需要更可靠的长时期持久性，请考虑使用 PostgreSQL 或 MySQL（通过 SQLAlchemy）、Cassandra 或自定义定义的后端。*

### SQS

SQS 是一个代理（broker）。

如果您已经与 AWS 紧密集成，并且熟悉 SQS，它作为一个代理是一个很好的选择。它极其可扩展且完全托管，任务委派方式与 RabbitMQ 类似。但它缺少 RabbitMQ 代理的一些功能，例如 `worker remote control commands`。

### SQLAlchemy

SQLAlchemy 是一个后端（backend）。

它允许 Celery 与 MySQL、PostgreSQL、SQLite 等接口。它是一个 ORM，是 Celery 可以使用 SQL 数据库作为结果后端的方式。

### GCPubSub

Google Cloud Pub/Sub 是一个代理（broker）。

如果您已经与 Google Cloud 紧密集成，并且熟悉 Pub/Sub，它作为一个代理是一个很好的选择。它极其可扩展且完全托管，任务委派方式与 RabbitMQ 类似。
