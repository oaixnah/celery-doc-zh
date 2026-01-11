---
subtitle: Debugging
description: Celery远程调试指南：使用rdb模块进行任务调试，支持pdb断点设置、telnet远程连接和信号触发调试功能。
---

# 调试

## 远程调试任务（使用 pdb）

### 基础

`celery.contrib.rdb` 是 `pdb` 的扩展版本，它允许对没有终端访问权限的进程进行远程调试。

使用示例：

```python
from celery import task
from celery.contrib import rdb

@task()
def add(x, y):
    result = x + y
    rdb.set_trace()  # <- 设置断点
    return result
```

`celery.contrib.rdb.set_trace()` 在当前位置设置断点，并创建一个可以通过 telnet 连接的 socket，以便远程调试你的任务。

调试器可能会被多个进程同时启动，因此调试器不会使用固定端口，而是从基础端口（默认为 6900）开始搜索可用端口。可以通过环境变量 `CELERY_RDB_PORT` 更改基础端口。

默认情况下，调试器只能从本地主机访问，要启用外部访问，必须设置环境变量 `CELERY_RDB_HOST`。

当工作进程遇到你的断点时，它将记录以下信息：

```text
[INFO/MainProcess] Received task:
    tasks.add[d7261c71-4962-47e5-b342-2448bedd20e8]
[WARNING/PoolWorker-1] Remote Debugger:6900:
    Please telnet 127.0.0.1 6900.  Type `exit` in session to continue.
[2011-01-18 14:25:44,119: WARNING/PoolWorker-1] Remote Debugger:6900:
    Waiting for client...
```

如果你 telnet 到指定的端口，你将看到一个 `pdb` shell：

```console
$ telnet localhost 6900
Connected to localhost.
Escape character is '^]'.
> /opt/devel/demoapp/tasks.py(128)add()
-> return result
(Pdb)
```

输入 `help` 获取可用命令列表，如果你从未使用过 `pdb`，阅读 [Python 调试器手册](http://docs.python.org/library/pdb.html) 可能是个好主意。

为了演示，我们将读取 `result` 变量的值，更改它并继续执行任务：

```text
(Pdb) result
4
(Pdb) result = 'hello from rdb'
(Pdb) continue
Connection closed by foreign host.
```

我们的修改结果可以在工作进程日志中看到：

```text
[2011-01-18 14:35:36,599: INFO/MainProcess] Task
    tasks.add[d7261c71-4962-47e5-b342-2448bedd20e8] succeeded
    in 61.481s: 'hello from rdb'
```

### 提示

#### 启用断点信号

如果设置了环境变量 `CELERY_RDBSIG`，工作进程将在每次发送 `SIGUSR2` 信号时打开一个 rdb 实例。这适用于主进程和工作进程。

例如，使用以下命令启动工作进程：

```console
CELERY_RDBSIG=1 celery worker -l INFO
```

你可以通过执行以下命令为任何工作进程启动 rdb 会话：

```console
kill -USR2 <pid>
```
