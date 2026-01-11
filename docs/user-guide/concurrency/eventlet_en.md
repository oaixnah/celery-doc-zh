# Concurrency with Eventlet

## Introduction

The [Eventlet](http://eventlet.net) homepage describes it as
a concurrent networking library for Python that allows you to
change how you run your code, not how you write it.

- It uses [`epoll(4)`](http://linux.die.net/man/4/epoll) or [`libevent`](https://libevent.org) for `highly scalable non-blocking I/O`_.
- [`Coroutines`](https://en.wikipedia.org/wiki/Coroutine) ensure that the developer uses a blocking style of programming that's similar to threading, but provide the benefits of non-blocking I/O.
- The event dispatch is implicit: meaning you can easily use Eventlet from the Python interpreter, or as a small part of a larger application.

Celery supports Eventlet as an alternative execution pool implementation and
in some cases superior to prefork. However, you need to ensure one task doesn't
block the event loop too long. Generally, CPU-bound operations don't go well
with Eventlet. Also note that some libraries, usually with C extensions,
cannot be monkeypatched and therefore cannot benefit from using Eventlet.
Please refer to their documentation if you are not sure. For example, pylibmc
does not allow cooperation with Eventlet but psycopg2 does when both of them
are libraries with C extensions.


The prefork pool can take use of multiple processes, but how many is
often limited to a few processes per CPU. With Eventlet you can efficiently
spawn hundreds, or thousands of green threads. In an informal test with a
feed hub system the Eventlet pool could fetch and process hundreds of feeds
every second, while the prefork pool spent 14 seconds processing 100
feeds. Note that this is one of the applications async I/O is especially good
at (asynchronous HTTP requests). You may want a mix of both Eventlet and
prefork workers, and route tasks according to compatibility or
what works best.

## Enabling Eventlet

You can enable the Eventlet pool by using the :option:`celery worker -P`
worker option.

```console
celery -A proj worker -P eventlet -c 1000
```

## Examples

See the [Eventlet examples](https://github.com/celery/celery/tree/main/examples/eventlet) directory in the Celery distribution for
some examples taking use of Eventlet support.
