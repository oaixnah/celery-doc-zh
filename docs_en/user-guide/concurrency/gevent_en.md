# Concurrency with gevent

## Introduction

The [gevent](http://www.gevent.org/) homepage describes it a [coroutine](https://en.wikipedia.org/wiki/Coroutine)-based [Python](https://www.python.org/) networking library that uses
[greenlet](https://greenlet.readthedocs.io) to provide a high-level synchronous API on top of the [libev](http://libev.schmorp.de/)
or [libuv](http://libuv.org/) event loop.

Features include:

- Fast event loop based on [libev](http://libev.schmorp.de/) or [libuv](http://libuv.org/).
- Lightweight execution units based on greenlets.
- API that reuses concepts from the Python standard library (for examples there are [events](http://www.gevent.org/api/gevent.event.html#gevent.event.Event) and [queues](http://www.gevent.org/api/gevent.queue.html#gevent.queue.Queue)).
- [Cooperative sockets with SSL support](http://www.gevent.org/api/index.html#networking)
- [Cooperative DNS queries](http://www.gevent.org/dns.html) performed through a threadpool, dnspython, or c-ares.
- [Monkey patching utility](http://www.gevent.org/intro.html#monkey-patching) to get 3rd party modules to become cooperative
- TCP/UDP/HTTP servers
- Subprocess support (through [gevent.subprocess](http://www.gevent.org/api/gevent.subprocess.html#module-gevent.subprocess))
- Thread pools

gevent is [inspired by eventlet](http://blog.gevent.org/2010/02/27/why-gevent/) but features a more consistent API,
simpler implementation and better performance. Read why others [use gevent](http://groups.google.com/group/gevent/browse_thread/thread/4de9703e5dca8271)
and check out the list of the [open source projects based on gevent](https://github.com/gevent/gevent/wiki/Projects).

## Enabling gevent

You can enable the gevent pool by using the
:option:`celery worker -P gevent` or  :option:`celery worker --pool=gevent`
worker option.

```console
celery -A proj worker -P gevent -c 1000
```

## Examples

See the [gevent examples](https://github.com/celery/celery/tree/main/examples/gevent) directory in the Celery distribution for
some examples taking use of Eventlet support.

## Known issues

There is a known issue using python 3.11 and gevent.
The issue is documented [here](https://github.com/celery/celery/issues/8425) and addressed in a [gevent issue](https://github.com/gevent/gevent/issues/1985).
Upgrading to greenlet 3.0 solves it.
