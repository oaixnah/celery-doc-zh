---
subtitle: Introduction
---

# Introduction

## What's a Task Queue?

Task queues are used as a mechanism to distribute work across threads or
machines.

A task queue's input is a unit of work called a task. Dedicated worker
processes constantly monitor task queues for new work to perform.

Celery communicates via messages, usually using a broker
to mediate between clients and workers. To initiate a task the client adds a
message to the queue, the broker then delivers that message to a worker.

A Celery system can consist of multiple workers and brokers, giving way
to high availability and horizontal scaling.

Celery is written in Python, but the protocol can be implemented in any
language. In addition to Python there's node-celery_ for Node.js,
a [PHP client](https://github.com/gjedeer/celery-php), [gocelery](https://github.com/gocelery/gocelery), [gopher-celery](https://github.com/marselester/gopher-celery), and [rusty-celery](https://github.com/rusty-celery/rusty-celery).

Language interoperability can also be achieved
exposing an HTTP endpoint and having a task that requests it (webhooks).

## What do I need?

??? warning "Version Requirements"

    Celery version 5.5.x runs on:
    
    - Python ❨3.8, 3.9, 3.10, 3.11, 3.12, 3.13❩
    - PyPy3.9+ ❨v7.3.12+❩
    
    If you're running an older version of Python, you need to be running
    an older version of Celery:
    
    - Python 3.7: Celery 5.2 or earlier.
    - Python 3.6: Celery 5.1 or earlier.
    - Python 2.7: Celery 4.x series.
    - Python 2.6: Celery series 3.1 or earlier.
    - Python 2.5: Celery series 3.0 or earlier.
    - Python 2.4: Celery series 2.2 or earlier..
    
    Celery is a project with minimal funding,
    so we don't support Microsoft Windows.
    Please don't open any issues related to that platform.

*Celery* requires a message transport to send and receive messages.
The RabbitMQ and Redis broker transports are feature complete,
but there's also support for a myriad of other experimental solutions, including
using SQLite for local development.

*Celery* can run on a single machine, on multiple machines, or even
across data centers.

## Get Started

If this is the first time you're trying to use Celery, or if you haven't
kept up with development in the 3.1 version and are coming from previous versions,
then you should read our getting started tutorials:

- :ref:`first-steps`
- :ref:`next-steps`

## Celery is…

### Simple

Celery is easy to use and maintain, and it *doesn't need configuration files*.

Here's one of the simplest applications you can make:

```python
from celery import Celery

app = Celery('hello', broker='amqp://guest@localhost//')

@app.task
def hello():
    return 'hello world'
```

### Highly Available

Workers and clients will automatically retry in the event
of connection loss or failure, and some brokers support
HA in way of *Primary/Primary* or *Primary/Replica* replication.

### Fast

A single Celery process can process millions of tasks a minute,
with sub-millisecond round-trip latency (using RabbitMQ,
librabbitmq, and optimized settings).

### Flexible

Almost every part of *Celery* can be extended or used on its own,
Custom pool implementations, serializers, compression schemes, logging,
schedulers, consumers, producers, broker transports, and much more.


## It supports

### Brokers

- :ref:`RabbitMQ <broker-rabbitmq>`, :ref:`Redis <broker-redis>`,
- :ref:`Amazon SQS <broker-sqs>`, and more…

### Concurrency

- prefork (multiprocessing),
- Eventlet_, gevent_
- thread (multithreaded)
- `solo` (single threaded)

### Result Stores

- AMQP, Redis
- Memcached,
- SQLAlchemy, Django ORM
- Apache Cassandra, Elasticsearch, Riak
- MongoDB, CouchDB, Couchbase, ArangoDB
- Amazon DynamoDB, Amazon S3
- Microsoft Azure Block Blob, Microsoft Azure Cosmos DB
- Google Cloud Storage
- File system

### Serialization

- *pickle*, *json*, *yaml*, *msgpack*.
- *zlib*, *bzip2* compression.
- Cryptographic message signing.

## Features

### Monitoring

A stream of monitoring events is emitted by workers and
is used by built-in and external tools to tell you what
your cluster is doing -- in real-time.

:ref:`Read more… <guide-monitoring>`.

### Work-flows

Simple and complex work-flows can be composed using
a set of powerful primitives we call the "canvas",
including grouping, chaining, chunking, and more.

:ref:`Read more… <guide-canvas>`.

### Time & Rate Limits

You can control how many tasks can be executed per second/minute/hour,
or how long a task can be allowed to run, and this can be set as
a default, for a specific worker or individually for each task type.

:ref:`Read more… <worker-time-limits>`.

### Scheduling

You can specify the time to run a task in seconds or a
:class:`~datetime.datetime`, or you can use
periodic tasks for recurring events based on a
simple interval, or Crontab expressions
supporting minute, hour, day of week, day of month, and
month of year.

:ref:`Read more… <guide-beat>`.

### Resource Leak Protection

The :option:`--max-tasks-per-child <celery worker --max-tasks-per-child>`
option is used for user tasks leaking resources, like memory or
file descriptors, that are simply out of your control.

:ref:`Read more… <worker-max-tasks-per-child>`.

### User Components

Each worker component can be customized, and additional components
can be defined by the user. The worker is built up using "bootsteps" — a
dependency graph enabling fine grained control of the worker's
internals.

.. _`Eventlet`: http://eventlet.net/
.. _`gevent`: http://gevent.org/

## Framework Integration

Celery is easy to integrate with web frameworks, some of them even have
integration packages:

+--------------------+------------------------+
| `Pyramid`_         | :pypi:`pyramid_celery` |
+--------------------+------------------------+
| `Pylons`_          | :pypi:`celery-pylons`  |
+--------------------+------------------------+
| `Flask`_           | not needed             |
+--------------------+------------------------+
| `web2py`_          | :pypi:`web2py-celery`  |
+--------------------+------------------------+
| `Tornado`_         | :pypi:`tornado-celery` |
+--------------------+------------------------+
| `Tryton`_          | :pypi:`celery_tryton`  |
+--------------------+------------------------+

For `Django`_ see :ref:`django-first-steps`.

The integration packages aren't strictly necessary, but they can make
development easier, and sometimes they add important hooks like closing
database connections at :manpage:`fork(2)`.

.. _`Django`: https://djangoproject.com/
.. _`Pylons`: http://pylonshq.com/
.. _`Flask`: http://flask.pocoo.org/
.. _`web2py`: http://web2py.com/
.. _`Bottle`: https://bottlepy.org/
.. _`Pyramid`: http://docs.pylonsproject.org/en/latest/docs/pyramid.html
.. _`Tornado`: http://www.tornadoweb.org/
.. _`Tryton`: http://www.tryton.org/
.. _`tornado-celery`: https://github.com/mher/tornado-celery/
