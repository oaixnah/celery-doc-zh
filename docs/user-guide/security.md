---
subtitle: Security
description: Celery分布式任务队列安全指南 - 涵盖代理保护、客户端安全、工作进程权限控制、序列化器安全配置、消息签名认证以及入侵检测策略，帮助您构建安全的Celery应用部署环境。
---

# 安全

## 介绍

虽然 Celery 在设计时考虑了安全性，但仍应将其视为不安全的组件。

根据您的[安全策略](https://en.wikipedia.org/wiki/Security_policy)，可以采取各种步骤来使您的 Celery 安装更加安全。

## 关注领域

### 代理 (Broker)

必须保护代理免受不必要的访问，尤其是在可公开访问的情况下。默认情况下，工作进程信任从代理获取的数据未被篡改。

第一道防线应该是在代理前面设置防火墙，只允许白名单中的机器访问它。

请记住，防火墙配置错误和临时禁用防火墙在现实世界中都很常见。健全的安全策略包括监控防火墙设备以检测它们是否被禁用，无论是意外还是故意。

换句话说，也不应盲目信任防火墙。

如果您的代理支持细粒度访问控制，例如 RabbitMQ，您应该考虑启用此功能。例如，请参阅 http://www.rabbitmq.com/access-control.html。

如果您的代理后端支持，可以使用 `broker_use_ssl` 启用端到端 SSL 加密和身份验证。

### 客户端 (Client)

在 Celery 中，“客户端”指的是任何向代理发送消息的东西，例如应用任务的 Web 服务器。

如果可以通过客户端发送任意消息，那么正确保护代理就没有意义。

### 工作进程 (Worker)

在工作进程内部运行的任务的默认权限与工作进程本身的权限相同。这适用于资源，例如：内存、文件系统和设备。

此规则的一个例外是使用基于多进程的任务池（当前为默认设置）。在这种情况下，任务将有权访问由于 `fork()` 调用而复制的任何内存，以及在同一工作进程子进程中由父任务写入的内存内容。

可以通过在每个任务中启动子进程（`fork()` + `execve()`）来限制对内存内容的访问。

可以通过使用 [chroot](https://en.wikipedia.org/wiki/Chroot)、[jail](https://en.wikipedia.org/wiki/FreeBSD_jail)、[沙盒](https://en.wikipedia.org/wiki/Sandbox_(computer_security))、虚拟机或平台或附加软件启用的其他机制来限制文件系统和设备访问。

还要注意，在工作进程中执行的任何任务都将具有与其运行的机器相同的网络访问权限。如果工作进程位于内部网络上，建议为出站流量添加防火墙规则。

## 序列化器 (Serializers)

自版本 4.0 起，默认的序列化器是 JSON，但由于它仅支持有限的一组类型，您可能需要考虑使用 pickle 进行序列化。

`pickle` 序列化器很方便，因为它可以序列化几乎任何 Python 对象，甚至经过一些处理后可以序列化函数，但出于同样的原因，`pickle` 本质上是不安全的，应避免在客户端不受信任或未经验证时使用。

您可以通过在 `accept_content` 设置中指定接受的内容类型的白名单来禁用不受信任的内容：

!!! note

    此设置在版本 3.0.18 中首次支持。如果您运行的是早期版本，它将被忽略，因此请确保您运行的版本支持它。

```python
accept_content = ['json']
```

这接受序列化器名称和内容类型的列表，因此您也可以为 json 指定内容类型：

```python
accept_content = ['application/json']
```

Celery 还附带了一个特殊的 `auth` 序列化器，用于验证 Celery 客户端和工作进程之间的通信，确保消息来自可信来源。
使用[公钥密码学](https://en.wikipedia.org/wiki/Public-key_cryptography)，`auth` 序列化器可以验证发送者的真实性。

## 消息签名

Celery 可以使用 `cryptography` 库通过 `公钥密码学` 来签名消息，其中客户端发送的消息使用私钥签名，然后由工作进程使用公钥证书进行验证。

最佳情况下，证书应由官方的[证书颁发机构](https://en.wikipedia.org/wiki/Certificate_authority)签名，但也可以使用自签名证书。

要启用此功能，您应将 `task_serializer` 设置配置为使用 `auth` 序列化器。要强制工作进程仅接受签名消息，应将 `accept_content` 设置为 `['auth']`。
对于事件协议的额外签名，将 `event_serializer` 设置为 `auth`。
还需要配置文件系统中用于定位私钥和证书的路径：分别是 `security_key`、`security_certificate` 和 `security_cert_store` 设置。
您可以使用 `security_digest` 调整签名算法。
如果使用加密的私钥，可以通过 `security_key_password` 配置密码。

配置这些后，还需要调用 `setup_security()` 函数。请注意，这也会禁用所有不安全的序列化器，以便工作进程不会接受不受信任内容类型的消息。

这是一个使用 `auth` 序列化器的配置示例，私钥和证书文件位于 `/etc/ssl` 目录中。

```python
app = Celery()
app.conf.update(
    security_key='/etc/ssl/private/worker.key'
    security_certificate='/etc/ssl/certs/worker.pem'
    security_cert_store='/etc/ssl/certs/*.pem',
    security_digest='sha256',
    task_serializer='auth',
    event_serializer='auth',
    accept_content=['auth']
)
app.setup_security()
```

!!! note

    虽然不禁止使用相对路径，但建议对这些文件使用绝对路径。

    另请注意，`auth` 序列化器不会加密消息内容，因此如果需要，必须单独启用加密功能。

## 入侵检测

在防御系统免受入侵者攻击时，最重要的部分是能够检测系统是否已被入侵。

### 日志

日志通常是查找安全漏洞证据的第一个地方，但如果日志可以被篡改，它们就毫无用处。

一个好的解决方案是设置带有专用日志服务器的集中式日志记录。应限制对其的访问。除了将所有日志放在一个地方之外，如果配置正确，它可以使入侵者更难篡改您的日志。

使用 syslog（另请参阅 [syslog-ng](https://en.wikipedia.org/wiki/Syslog-ng) 和 [rsyslog](http://www.rsyslog.com/)）应该相当容易设置。Celery 使用 `logging` 库，并且已经支持使用 syslog。

对于偏执狂的一个提示是使用 UDP 发送日志，并切断日志服务器的网络电缆的传输部分 :-)

### Tripwire

[Tripwire](http://tripwire.com/) 是一个（现在是商业的）数据完整性工具，有多个开源实现，用于在文件系统中保存文件的加密哈希值，以便在文件更改时向管理员发出警报。这样，当损害已经造成并且您的系统已被入侵时，您可以准确知道入侵者更改了哪些文件（密码文件、日志、后门、rootkit 等）。通常这是您能够检测到入侵的唯一方法。

一些开源实现包括：

* [OSSEC](http://www.ossec.net/)
* [Samhain](http://la-samhna.de/samhain/index.html)
* [Open Source Tripwire](https://github.com/Tripwire/tripwire-open-source)
* [AIDE](http://aide.sourceforge.net/)

此外，[ZFS](https://en.wikipedia.org/wiki/ZFS) 文件系统带有内置的完整性检查功能，可以使用。