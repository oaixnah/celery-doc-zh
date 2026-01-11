---
subtitle: Daemonization
description: Celery守护进程配置指南 - 详细讲解如何在Linux系统中使用systemd和init.d脚本将Celery worker和beat服务配置为守护进程，包括完整的配置文件示例、故障排除方法和安全注意事项。
---

# 守护进程

如今大多数 Linux 发行版都使用 systemd 来管理系统和用户服务的生命周期。

您可以通过输入以下命令来检查您的 Linux 发行版是否使用 systemd：

```console
systemctl --version
systemd 249 (v249.9-1.fc35)
+PAM +AUDIT +SELINUX -APPARMOR +IMA +SMACK +SECCOMP +GCRYPT +GNUTLS +OPENSSL +ACL +BLKID +CURL +ELFUTILS +FIDO2 +IDN2 -IDN +IPTC +KMOD +LIBCRYPTSETUP +LIBFDISK +PCRE2 +PWQUALITY +P11KIT +QRENCODE +BZIP2 +LZ4 +XZ +ZLIB +ZSTD +XKBCOMMON +UTMP +SYSVINIT default-hierarchy=unified
```

如果您有类似的输出，请参考 [使用 systemd](#daemon-systemd-generic) 获取指导。

然而，init.d 脚本在这些 Linux 发行版中仍然应该有效，因为 systemd 提供了 systemd-sysv 兼容层，它会自动从我们提供的 init.d 脚本生成服务。

如果您为多个 Linux 发行版打包 Celery，并且其中一些不支持 systemd，或者也面向其他 Unix 系统，您可能需要参考 [通用初始化脚本](#daemon-generic)。

## 通用初始化脚本 {#daemon-generic}

请参阅 Celery 发行版中的 [extra/generic-init.d/](https://github.com/celery/celery/tree/main/extra/generic-init.d/){target="_blank"} 目录。

该目录包含用于 `celery worker` 程序的通用 bash 初始化脚本，这些脚本应该可以在 Linux、FreeBSD、OpenBSD 和其他类 Unix 平台上运行。

### 初始化脚本: `celeryd`

**用法**: `/etc/init.d/celeryd {start|stop|restart|status}`

**配置文件**: `/etc/default/celeryd`

要正确配置此脚本以运行 worker，您至少需要告诉它在启动时切换到哪个目录（以找到包含您的应用程序或配置模块的模块）。

守护进程脚本由文件 `/etc/default/celeryd` 配置。这是一个 shell (`sh`) 脚本，您可以在其中添加环境变量，如下面的配置选项所示。要添加影响 worker 的实际环境变量，您还必须导出它们（例如，`export DISPLAY=":0"`）

!!! tip "需要超级用户权限"

    初始化脚本只能由 root 用户使用，并且 shell 配置文件也必须归 root 所有。

    非特权用户不需要使用初始化脚本，而是可以使用 `celery multi` 实用程序（或 `celery worker --detach`）：

    ```console
    celery -A proj multi start worker1 \
        --pidfile="$HOME/run/celery/%n.pid" \
        --logfile="$HOME/log/celery/%n%I.log"

    celery -A proj multi restart worker1 \
        --logfile="$HOME/log/celery/%n%I.log" \
        --pidfile="$HOME/run/celery/%n.pid

    celery multi stopwait worker1 --pidfile="$HOME/run/celery/%n.pid"
    ```

#### 示例配置

这是一个 Python 项目的示例配置。

```bash title="/etc/default/celeryd"
# 要启动的节点名称
#   大多数人只会启动一个节点：
CELERYD_NODES="worker1"
#   但您也可以启动多个节点并在 CELERYD_OPTS 中为每个节点配置设置
#CELERYD_NODES="worker1 worker2 worker3"
#   或者，您可以指定要启动的节点数量：
#CELERYD_NODES=10

# 'celery' 命令的绝对或相对路径：
CELERY_BIN="/usr/local/bin/celery"
#CELERY_BIN="/virtualenvs/def/bin/celery"

# 要使用的应用程序实例
# 如果不使用应用程序，请注释掉此行
CELERY_APP="proj"
# 或完全限定：
#CELERY_APP="proj.tasks:app"

# 启动时要切换到的目录。
CELERYD_CHDIR="/opt/Myproject/"

# 传递给 worker 的额外命令行参数
CELERYD_OPTS="--time-limit=300 --concurrency=8"
# 通过将节点名称附加到参数来配置特定于节点的设置：
#CELERYD_OPTS="--time-limit=300 -c 8 -c:worker2 4 -c:worker3 2 -Ofair:worker1"

# 将日志级别设置为 DEBUG
#CELERYD_LOG_LEVEL="DEBUG"

# %n 将被替换为节点名称的第一部分。
CELERYD_LOG_FILE="/var/log/celery/%n%I.log"
CELERYD_PID_FILE="/var/run/celery/%n.pid"

# Worker 应该以非特权用户身份运行。
#   您需要手动创建此用户（或者您可以选择
#   已经存在的用户/组组合（例如，nobody））。
CELERYD_USER="celery"
CELERYD_GROUP="celery"

# 如果启用，将在缺失时创建 pid 和日志目录，
# 并由配置的用户 ID/组拥有。
CELERY_CREATE_DIRS=1
```

### 使用登录 shell

您可以通过使用登录 shell 来继承 `CELERYD_USER` 的环境：

```bash
CELERYD_SU_ARGS="-l"
```

请注意，这不被推荐，您应该只在绝对必要时使用此选项。

### Django 示例配置

Django 用户现在使用与上面完全相同的模板，但请确保定义 Celery 应用程序实例的模块也为 `DJANGO_SETTINGS_MODULE` 设置了默认值，如 :ref:`django-first-steps` 中的示例 Django 项目所示。

### 可用选项

<table>
<thead>
<tr>
    <th>选项</th>
    <th>描述</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>CELERY_APP</code></td>
<td>
要使用的应用程序实例（<code>celery --app</code> 参数的值）。
</td>
</tr>
<tr>
<td><code>CELERY_BIN</code></td>
<td>
<code>celery</code> 程序的绝对或相对路径。
示例：

<code>celery</code>
<code>/usr/local/bin/celery</code>
<code>/virtualenvs/proj/bin/celery</code>
<code>/virtualenvs/proj/bin/python -m celery</code>
</td>
</tr>
<tr>
<td><code>CELERYD_NODES</code></td>
<td>
要启动的节点名称列表（用空格分隔）。
</td>
</tr>
<tr>
<td><code>CELERYD_OPTS</code></td>
<td>
worker 的额外命令行参数，请参阅
<code>celery worker --help</code> 获取列表。这也支持 <code>multi</code> 使用的扩展语法
来配置各个节点的设置。
请参阅 <code>celery multi --help</code> 获取一些多节点配置示例。
</td>
</tr>
<tr>
<td><code>CELERYD_CHDIR</code></td>
<td>
启动时要切换到的路径。默认是保持在当前目录。
</td>
</tr>
<tr>
<td><code>CELERYD_PID_FILE</code></td>
<td>
PID 文件的完整路径。默认是 /var/run/celery/%n.pid
</td>
</tr>
<tr>
<td><code>CELERYD_LOG_FILE</code></td>
<td>
worker 日志文件的完整路径。默认是 /var/log/celery/%n%I.log
**注意**：在使用 prefork 池时，使用 <code>%I</code> 很重要，因为
多个进程共享同一个日志文件会导致竞争条件。
</td>
</tr>
<tr>
<td><code>CELERYD_LOG_LEVEL</code></td>
<td>
worker 日志级别。默认是 INFO。
</td>
</tr>
<tr>
<td><code>CELERYD_USER</code></td>
<td>
运行 worker 的用户。默认是当前用户。
</td>
</tr>
<tr>
<td><code>CELERYD_GROUP</code></td>
<td>
运行 worker 的组。默认是当前用户。
</td>
</tr>
<tr>
<td><code>CELERY_CREATE_DIRS</code></td>
<td>
始终创建目录（日志目录和 pid 文件目录）。
默认仅在未设置自定义日志文件/pid 文件时创建目录。
</td>
</tr>
<tr>
<td><code>CELERY_CREATE_RUNDIR</code></td>
<td>
始终创建 pid 文件目录。默认仅在未设置自定义 pid 文件位置时启用。
</td>
</tr>
<tr>
<td><code>CELERY_CREATE_LOGDIR</code></td>
<td>
始终创建日志文件目录。默认仅在未设置自定义日志文件位置时启用。
</td>
</tr>
</tbody>
</table>

### 初始化脚本: `celerybeat`

**用法**: `/etc/init.d/celerybeat {start|stop|restart}`

**配置文件**: `/etc/default/celerybeat` 或 `/etc/default/celeryd`。

#### 示例配置

这是一个 Python 项目的示例配置：

```bash title="/etc/default/celerybeat"
# 'celery' 命令的绝对或相对路径：
CELERY_BIN="/usr/local/bin/celery"
#CELERY_BIN="/virtualenvs/def/bin/celery"

# 要使用的应用程序实例
# 如果不使用应用程序，请注释掉此行
CELERY_APP="proj"
# 或完全限定：
#CELERY_APP="proj.tasks:app"

# 启动时要切换到的目录。
CELERYBEAT_CHDIR="/opt/Myproject/"

# 传递给 celerybeat 的额外参数
CELERYBEAT_OPTS="--schedule=/var/run/celery/celerybeat-schedule"
```

#### Django 示例配置

您应该使用与上面相同的模板，但请确保 `DJANGO_SETTINGS_MODULE` 变量已设置（并导出），并且 `CELERYD_CHDIR` 设置为项目目录：

```bash
export DJANGO_SETTINGS_MODULE="settings"

CELERYD_CHDIR="/opt/MyProject"
```

#### 可用选项

| 选项                     | 描述                                                         |
|------------------------|------------------------------------------------------------|
| `CELERY_APP`           | 要使用的应用程序实例（`celery --app` 参数的值）。                           |
| `CELERYBEAT_OPTS`      | 传递给 `celery beat` 的额外参数，请参阅 `celery beat --help` 获取可用选项列表。 |
| `CELERYBEAT_PID_FILE`  | PID 文件的完整路径。默认是 `/var/run/celeryd.pid`。                    |
| `CELERYBEAT_LOG_FILE`  | 日志文件的完整路径。默认是 `/var/log/celeryd.log`。                      |
| `CELERYBEAT_LOG_LEVEL` | 要使用的日志级别。默认是 `INFO`。                                       |
| `CELERYBEAT_USER`      | 运行 beat 的用户。默认是当前用户。                                       |
| `CELERYBEAT_GROUP`     | 运行 beat 的组。默认是当前用户。                                        |
| `CELERY_CREATE_DIRS`   | 始终创建目录（日志目录和 pid 文件目录）。默认仅在未设置自定义日志文件/pid 文件时创建目录。         |
| `CELERY_CREATE_RUNDIR` | 始终创建 pid 文件目录。默认仅在未设置自定义 pid 文件位置时启用。                      |
| `CELERY_CREATE_LOGDIR` | 始终创建日志文件目录。默认仅在未设置自定义日志文件位置时启用。                            |

### 故障排除

如果无法使初始化脚本正常工作，您应该尝试在*详细模式*下运行它们：

```console
sh -x /etc/init.d/celeryd start
```

这可以揭示服务无法启动的原因。

如果 worker 以 *"OK"* 启动但几乎立即退出，并且日志文件中没有证据，那么可能存在错误，但由于守护进程的标准输出已经关闭，您将无法在任何地方看到它们。对于这种情况，您可以使用 `C_FAKEFORK` 环境变量跳过守护进程步骤：

```console
C_FAKEFORK=1 sh -x /etc/init.d/celeryd start
```

现在您应该能够看到错误了。

通常，此类错误是由文件读写权限不足、配置模块、用户模块、第三方库中的语法错误，甚至 Celery 本身引起的。

## 使用 `systemd` {#daemon-systemd-generic}

**用法**: `systemctl {start|stop|restart|status} celery.service`

**配置文件**: `/etc/conf.d/celery`

### 服务文件: celery.service

这是一个示例的systemd文件：

```bash title="/etc/systemd/system/celery.service"
[Unit]
Description=Celery Service
After=network.target

[Service]
Type=forking
User=celery
Group=celery
EnvironmentFile=/etc/conf.d/celery
WorkingDirectory=/opt/celery
ExecStart=/bin/sh -c '${CELERY_BIN} -A $CELERY_APP multi start $CELERYD_NODES \
    --pidfile=${CELERYD_PID_FILE} --logfile=${CELERYD_LOG_FILE} \
    --loglevel="${CELERYD_LOG_LEVEL}" $CELERYD_OPTS'
ExecStop=/bin/sh -c '${CELERY_BIN} multi stopwait $CELERYD_NODES \
    --pidfile=${CELERYD_PID_FILE} --logfile=${CELERYD_LOG_FILE} \
    --loglevel="${CELERYD_LOG_LEVEL}"'
ExecReload=/bin/sh -c '${CELERY_BIN} -A $CELERY_APP multi restart $CELERYD_NODES \
    --pidfile=${CELERYD_PID_FILE} --logfile=${CELERYD_LOG_FILE} \
    --loglevel="${CELERYD_LOG_LEVEL}" $CELERYD_OPTS'
Restart=always

[Install]
WantedBy=multi-user.target
```

一旦你将该文件放入 `/etc/systemd/system`，你应该运行 `systemctl daemon-reload` 以便Systemd识别该文件。每次修改该文件时，你也应该运行该命令。如果你希望 celery 服务在系统（重新）启动时自动启动，请使用 `systemctl enable celery.service`。

可选地，你可以为celery服务指定额外的依赖项：例如，如果你使用 RabbitMQ 作为代理，你可以在 `[Unit]` [systemd部分](https://www.freedesktop.org/software/systemd/man/systemd.unit.html#%5BUnit%5D%20Section%20Options) 中的 `After=` 和 `Requires=` 中指定 `rabbitmq-server.service`。

要配置用户、组、`chdir` 更改设置：在 `/etc/systemd/system/celery.service` 中定义的 `User`、`Group` 和 `WorkingDirectory`。

你也可以使用 `systemd-tmpfiles` 来创建工作目录（用于日志和 pid 文件）。

```bash title="/etc/tmpfiles.d/celery.conf"
d /run/celery 0755 celery celery -
d /var/log/celery 0755 celery celery -
```

#### 示例配置

这是一个Python项目的示例配置：

```bash title="/etc/conf.d/celery"
# 要启动的节点名称
# 这里我们有一个单节点
CELERYD_NODES="w1"
# 或者我们可以有三个节点：
#CELERYD_NODES="w1 w2 w3"

# 'celery'命令的绝对或相对路径：
CELERY_BIN="/usr/local/bin/celery"
#CELERY_BIN="/virtualenvs/def/bin/celery"

# 要使用的应用实例
# 如果不使用应用，请注释掉这一行
CELERY_APP="proj"
# 或者完全限定：
#CELERY_APP="proj.tasks:app"

# 如何调用manage.py
CELERYD_MULTI="multi"

# 工作进程的额外命令行参数
CELERYD_OPTS="--time-limit=300 --concurrency=8"

# - %n 将被节点名称的第一部分替换。
# - %I 将被当前子进程索引替换
#   在使用prefork池时很重要，以避免竞争条件。
CELERYD_PID_FILE="/var/run/celery/%n.pid"
CELERYD_LOG_FILE="/var/log/celery/%n%I.log"
CELERYD_LOG_LEVEL="INFO"

# 你可能希望为Celery Beat添加这些选项
CELERYBEAT_PID_FILE="/var/run/celery/beat.pid"
CELERYBEAT_LOG_FILE="/var/log/celery/beat.log"
```

### 服务文件: celerybeat.service

这是一个Celery Beat的示例systemd文件：

```bash title="/etc/systemd/system/celerybeat.service"
[Unit]
Description=Celery Beat Service
After=network.target

[Service]
Type=simple
User=celery
Group=celery
EnvironmentFile=/etc/conf.d/celery
WorkingDirectory=/opt/celery
ExecStart=/bin/sh -c '${CELERY_BIN} -A ${CELERY_APP} beat  \
    --pidfile=${CELERYBEAT_PID_FILE} \
    --logfile=${CELERYBEAT_LOG_FILE} --loglevel=${CELERYD_LOG_LEVEL}'
Restart=always

[Install]
WantedBy=multi-user.target
```

一旦你将该文件放入 `/etc/systemd/system`，你应该运行 `systemctl daemon-reload` 以便 Systemd 识别该文件。每次修改该文件时，你也应该运行该命令。如果你希望 celery beat 服务在系统（重新）启动时自动启动，请使用 `systemctl enable celerybeat.service`。

## 使用超级用户权限（root）运行工作进程

使用超级用户权限运行工作进程是一种非常危险的做法。应该总是有避免以 root 身份运行的解决方案。Celery 可能会在通过 pickle 序列化的消息中运行任意代码 - 这是危险的，尤其是在以 root 身份运行时。

默认情况下，Celery不会以root身份运行工作进程。相关的错误消息可能不会在日志中可见，但如果使用了 `C_FAKEFORK` 则可能会看到。

要强制 Celery 以 root 身份运行工作进程，请使用 `C_FORCE_ROOT`。

当以 root 身份运行但没有 `C_FORCE_ROOT` 时，工作进程将显示以 *"OK"* 启动，但随后立即退出且无明显错误。这个问题可能会在新的开发或生产环境中（无意中）以root身份运行项目时出现。
