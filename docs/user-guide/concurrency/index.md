---
subtitle: Concurrency
description: Celery并发指南：详细介绍prefork、eventlet、gevent、solo、threads和custom等并发模型，帮助您选择最适合任务类型的worker池配置，优化任务执行性能。
---

# 并发

Celery 中的并发支持任务的并行执行。默认模型 `prefork` 适用于许多场景，通常推荐给大多数用户使用。事实上，切换到其他模式会静默禁用某些功能，如 `soft_timeout` 和 `max_tasks_per_child`。

本页简要概述了可用的选项，您可以在启动 worker 时使用 `--pool` 选项进行选择。

## 并发选项概述

- `prefork`：默认选项，适用于CPU密集型任务和大多数用例。它稳健可靠，除非有特定需求，否则推荐使用。
- `eventlet` 和 `gevent`：专为 IO 密集型任务设计，这些模型使用 greenlets 实现高并发。请注意，某些功能（如 `soft_timeout`）在这些模式下不可用。这些模式有详细的文档页面链接如下。
- `solo`：在主线程中顺序执行任务。
- `threads`：利用线程实现并发，如果 `concurrent.futures` 模块存在则可用。
- `custom`：允许通过环境变量指定自定义的worker池实现。

- [Eventlet](eventlet.md)
- [Gevent](gevent.md)

!!! note

    虽然 `eventlet` 和 `gevent` 等替代模型可用，但它们可能缺少 `prefork` 的某些功能。除非有特定要求，否则我们推荐将 `prefork` 作为起点。
