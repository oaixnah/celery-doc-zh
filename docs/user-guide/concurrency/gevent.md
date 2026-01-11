---
subtitle: gevent
description: Celery中使用gevent实现高并发处理的完整指南，包括gevent池的启用方法、配置示例、特性介绍以及Python 3.11兼容性问题的解决方案。
---

# 使用 gevent 的并发

## 介绍

[gevent](http://www.gevent.org/) 主页将其描述为一个基于[协程](https://en.wikipedia.org/wiki/Coroutine)的 [Python](https://www.python.org/) 网络库，它使用
[greenlet](https://greenlet.readthedocs.io) 在 [libev](http://libev.schmorp.de/) 或 [libuv](http://libuv.org/) 事件循环之上提供高级同步 API。

特性包括：

- 基于 [libev](http://libev.schmorp.de/) 或 [libuv](http://libuv.org/) 的快速事件循环
- 基于 greenlets 的轻量级执行单元
- 重用 Python 标准库概念的 API（例如有 [events](http://www.gevent.org/api/gevent.event.html#gevent.event.Event) 和 [queues](http://www.gevent.org/api/gevent.queue.html#gevent.queue.Queue)）
- [支持 SSL 的协作式套接字](http://www.gevent.org/api/index.html#networking)
- 通过线程池、dnspython 或 c-ares 执行的[协作式 DNS 查询](http://www.gevent.org/dns.html)
- [猴子补丁工具](http://www.gevent.org/intro.html#monkey-patching)使第三方模块变得协作式
- TCP/UDP/HTTP 服务器
- 子进程支持（通过 [gevent.subprocess](http://www.gevent.org/api/gevent.subprocess.html#module-gevent.subprocess)）
- 线程池

gevent [受 eventlet 启发](http://blog.gevent.org/2010/02/27/why-gevent/)，但具有更一致的 API、更简单的实现和更好的性能。阅读其他人[为什么使用 gevent](http://groups.google.com/group/gevent/browse_thread/thread/4de9703e5dca8271)并查看[基于 gevent 的开源项目列表](https://github.com/gevent/gevent/wiki/Projects)。

## 启用 gevent

您可以使用 `celery worker -P gevent` 或 `celery worker --pool=gevent` 工作器选项来启用 gevent 池。

```console
celery -A proj worker -P gevent -c 1000
```

## 示例

请参阅 Celery 发行版中的 [gevent 示例](https://github.com/celery/celery/tree/main/examples/gevent) 目录，了解一些使用 Eventlet 支持的示例。

## 已知问题

使用 Python 3.11 和 gevent 存在一个已知问题。该问题在[此处](https://github.com/celery/celery/issues/8425)有文档记录，并在 [gevent 问题](https://github.com/gevent/gevent/issues/1985) 中得到解决。升级到 greenlet 3.0 可以解决此问题。