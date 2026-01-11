# Daemonization

Most Linux distributions these days use systemd for managing the lifecycle of system
and user services.

You can check if your Linux distribution uses systemd by typing:

```console
systemctl --version
systemd 249 (v249.9-1.fc35)
+PAM +AUDIT +SELINUX -APPARMOR +IMA +SMACK +SECCOMP +GCRYPT +GNUTLS +OPENSSL +ACL +BLKID +CURL +ELFUTILS +FIDO2 +IDN2 -IDN +IPTC +KMOD +LIBCRYPTSETUP +LIBFDISK +PCRE2 +PWQUALITY +P11KIT +QRENCODE +BZIP2 +LZ4 +XZ +ZLIB +ZSTD +XKBCOMMON +UTMP +SYSVINIT default-hierarchy=unified
```

If you have output similar to the above, please refer to
:ref:`our systemd documentation <daemon-systemd-generic>` for guidance.

However, the init.d script should still work in those Linux distributions
as well since systemd provides the systemd-sysv compatibility layer
which generates services automatically from the init.d scripts we provide.

If you package Celery for multiple Linux distributions
and some do not support systemd or to other Unix systems as well,
you may want to refer to :ref:`our init.d documentation <daemon-generic>`.

## Generic init-scripts

See the `extra/generic-init.d/`_ directory Celery distribution.

This directory contains generic bash init-scripts for the
:program:`celery worker` program,
these should run on Linux, FreeBSD, OpenBSD, and other Unix-like platforms.

[extra/generic-init.d/](https://github.com/celery/celery/tree/main/extra/generic-init.d/)

### Init-script: `celeryd`

:Usage: `/etc/init.d/celeryd {start|stop|restart|status}`
:Configuration file: :file:`/etc/default/celeryd`

To configure this script to run the worker properly you probably need to at least
tell it where to change
directory to when it starts (to find the module containing your app, or your
configuration module).

The daemonization script is configured by the file :file:`/etc/default/celeryd`.
This is a shell (:command:`sh`) script where you can add environment variables like
the configuration options below.  To add real environment variables affecting
the worker you must also export them (e.g., :command:`export DISPLAY=":0"`)

!!! tip "Superuser privileges required"

    The init-scripts can only be used by root,
    and the shell configuration file must also be owned by root.

    Unprivileged users don't need to use the init-script,
    instead they can use the :program:`celery multi` utility (or
    :program:`celery worker --detach`):

    ```console
    celery -A proj multi start worker1 \
        --pidfile="$HOME/run/celery/%n.pid" \
        --logfile="$HOME/log/celery/%n%I.log"

    celery -A proj multi restart worker1 \
        --logfile="$HOME/log/celery/%n%I.log" \
        --pidfile="$HOME/run/celery/%n.pid

    celery multi stopwait worker1 --pidfile="$HOME/run/celery/%n.pid"
    ```

#### Example configuration

This is an example configuration for a Python project.

:file:`/etc/default/celeryd`:

```bash
# Names of nodes to start
#   most people will only start one node:
CELERYD_NODES="worker1"
#   but you can also start multiple and configure settings
#   for each in CELERYD_OPTS
#CELERYD_NODES="worker1 worker2 worker3"
#   alternatively, you can specify the number of nodes to start:
#CELERYD_NODES=10

# Absolute or relative path to the 'celery' command:
CELERY_BIN="/usr/local/bin/celery"
#CELERY_BIN="/virtualenvs/def/bin/celery"

# App instance to use
# comment out this line if you don't use an app
CELERY_APP="proj"
# or fully qualified:
#CELERY_APP="proj.tasks:app"

# Where to chdir at start.
CELERYD_CHDIR="/opt/Myproject/"

# Extra command-line arguments to the worker
CELERYD_OPTS="--time-limit=300 --concurrency=8"
# Configure node-specific settings by appending node name to arguments:
#CELERYD_OPTS="--time-limit=300 -c 8 -c:worker2 4 -c:worker3 2 -Ofair:worker1"

# Set logging level to DEBUG
#CELERYD_LOG_LEVEL="DEBUG"

# %n will be replaced with the first part of the nodename.
CELERYD_LOG_FILE="/var/log/celery/%n%I.log"
CELERYD_PID_FILE="/var/run/celery/%n.pid"

# Workers should run as an unprivileged user.
#   You need to create this user manually (or you can choose
#   a user/group combination that already exists (e.g., nobody).
CELERYD_USER="celery"
CELERYD_GROUP="celery"

# If enabled pid and log directories will be created if missing,
# and owned by the userid/group configured.
CELERY_CREATE_DIRS=1
```

### Using a login shell

You can inherit the environment of the `CELERYD_USER` by using a login
shell:

```bash
CELERYD_SU_ARGS="-l"
```

Note that this isn't recommended, and that you should only use this option
when absolutely necessary.

### Example Django configuration

Django users now uses the exact same template as above,
but make sure that the module that defines your Celery app instance
also sets a default value for :envvar:`DJANGO_SETTINGS_MODULE`
as shown in the example Django project in :ref:`django-first-steps`.

### Available options

<table>
<thead>
<tr>
    <th>Option</th>
    <th>Description</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>CELERY_APP</code></td>
<td>
App instance to use (value for :option:`--app <celery --app>` argument).
</td>
</tr>
<tr>
<td><code>CELERY_BIN</code></td>
<td>
Absolute or relative path to the :program:`celery` program.
Examples:

<code>celery</code>
<code>/usr/local/bin/celery</code>
<code>/virtualenvs/proj/bin/celery</code>
<code>/virtualenvs/proj/bin/python -m celery</code>
</td>
</tr>
<tr>
<td><code>CELERYD_NODES</code></td>
<td>
List of node names to start (separated by space).
</td>
</tr>
<tr>
<td><code>CELERYD_OPTS</code></td>
<td>
Additional command-line arguments for the worker, see
`celery worker --help` for a list. This also supports the extended
syntax used by `multi` to configure settings for individual nodes.
See `celery multi --help` for some multi-node configuration examples.
</td>
</tr>
<tr>
<td><code>CELERYD_CHDIR</code></td>
<td>
Path to change directory to at start. Default is to stay in the current
directory.
</td>
</tr>
<tr>
<td><code>CELERYD_PID_FILE</code></td>
<td>
Full path to the PID file. Default is /var/run/celery/%n.pid
</td>
</tr>
<tr>
<td><code>CELERYD_LOG_FILE</code></td>
<td>
Full path to the worker log file. Default is /var/log/celery/%n%I.log
**Note**: Using `%I` is important when using the prefork pool as having
multiple processes share the same log file will lead to race conditions.
</td>
</tr>
<tr>
<td><code>CELERYD_LOG_LEVEL</code></td>
<td>
Worker log level. Default is INFO.
</td>
</tr>
<tr>
<td><code>CELERYD_USER</code></td>
<td>
User to run the worker as. Default is current user.
</td>
</tr>
<tr>
<td><code>CELERYD_GROUP</code></td>
<td>
Group to run worker as. Default is current user.
</td>
</tr>
<tr>
<td><code>CELERY_CREATE_DIRS</code></td>
<td>
Always create directories (log directory and pid file directory).
Default is to only create directories when no custom logfile/pidfile set.
</td>
</tr>
<tr>
<td><code>CELERY_CREATE_RUNDIR</code></td>
<td>
Always create pidfile directory. By default only enabled when no custom
pidfile location set.
</td>
</tr>
<tr>
<td><code>CELERY_CREATE_LOGDIR</code></td>
<td>
Always create logfile directory. By default only enable when no custom
logfile location set.
</td>
</tr>
</tbody>
</table>

### Init-script: `celerybeat`

:Usage: `/etc/init.d/celerybeat {start|stop|restart}`
:Configuration file: :file:`/etc/default/celerybeat` or :file:`/etc/default/celeryd`.

#### Example configuration

This is an example configuration for a Python project:

`/etc/default/celerybeat`:

```bash
# Absolute or relative path to the 'celery' command:
CELERY_BIN="/usr/local/bin/celery"
#CELERY_BIN="/virtualenvs/def/bin/celery"

# App instance to use
# comment out this line if you don't use an app
CELERY_APP="proj"
# or fully qualified:
#CELERY_APP="proj.tasks:app"

# Where to chdir at start.
CELERYBEAT_CHDIR="/opt/Myproject/"

# Extra arguments to celerybeat
CELERYBEAT_OPTS="--schedule=/var/run/celery/celerybeat-schedule"
```

#### Example Django configuration

You should use the same template as above, but make sure the
`DJANGO_SETTINGS_MODULE` variable is set (and exported), and that
`CELERYD_CHDIR` is set to the projects directory:

```bash
export DJANGO_SETTINGS_MODULE="settings"

CELERYD_CHDIR="/opt/MyProject"
```

#### Available options

| Option                 | Description                                                                                                                                  |
|------------------------|----------------------------------------------------------------------------------------------------------------------------------------------|
| `CELERY_APP`           | App instance to use (value for :option:`--app <celery --app>` argument).                                                                     |
| `CELERYBEAT_OPTS`      | Additional arguments to :program:`celery beat`, see :command:`celery beat --help` for a list of available options.                           |
| `CELERYBEAT_PID_FILE`  | Full path to the PID file. Default is :file:`/var/run/celeryd.pid`.                                                                          |
| `CELERYBEAT_LOG_FILE`  | Full path to the log file. Default is :file:`/var/log/celeryd.log`.                                                                          |
| `CELERYBEAT_LOG_LEVEL` | Log level to use. Default is `INFO`.                                                                                                         |
| `CELERYBEAT_USER`      | User to run beat as. Default is the current user.                                                                                            |
| `CELERYBEAT_GROUP`     | Group to run beat as. Default is the current user.                                                                                           |
| `CELERY_CREATE_DIRS`   | Always create directories (log directory and pid file directory). Default is to only create directories when no custom logfile/pidfile set.  |
| `CELERY_CREATE_RUNDIR` | Always create pidfile directory. By default only enabled when no custom pidfile location set.                                                |
| `CELERY_CREATE_LOGDIR` | Always create logfile directory. By default only enable when no custom logfile location set.                                                 |

### Troubleshooting

If you can't get the init-scripts to work, you should try running
them in *verbose mode*:

```console
sh -x /etc/init.d/celeryd start
```

This can reveal hints as to why the service won't start.

If the worker starts with *"OK"* but exits almost immediately afterwards
and there's no evidence in the log file, then there's probably an error
but as the daemons standard outputs are already closed you'll
not be able to see them anywhere. For this situation you can use
the :envvar:`C_FAKEFORK` environment variable to skip the
daemonization step:

```console
C_FAKEFORK=1 sh -x /etc/init.d/celeryd start
```

and now you should be able to see the errors.

Commonly such errors are caused by insufficient permissions
to read from, or write to a file, and also by syntax errors
in configuration modules, user modules, third-party libraries,
or even from Celery itself (if you've found a bug you
should :ref:`report it <reporting-bugs>`).

## Usage `systemd`

[`extra/systemd/`](https://github.com/celery/celery/tree/main/extra/systemd/)

:Usage: `systemctl {start|stop|restart|status} celery.service`
:Configuration file: /etc/conf.d/celery

### Service file: celery.service

This is an example systemd file:

:file:`/etc/systemd/system/celery.service`:

```bash
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

Once you've put that file in :file:`/etc/systemd/system`, you should run
:command:`systemctl daemon-reload` in order that Systemd acknowledges that file.
You should also run that command each time you modify it.
Use :command:`systemctl enable celery.service` if you want the celery service to
automatically start when (re)booting the system.

Optionally you can specify extra dependencies for the celery service: e.g. if you use
RabbitMQ as a broker, you could specify `rabbitmq-server.service` in both `After=` and `Requires=`
in the `[Unit]` `systemd section <https://www.freedesktop.org/software/systemd/man/systemd.unit.html#%5BUnit%5D%20Section%20Options>`_.

To configure user, group, :command:`chdir` change settings:
`User`, `Group`, and `WorkingDirectory` defined in
:file:`/etc/systemd/system/celery.service`.

You can also use systemd-tmpfiles in order to create working directories (for logs and pid).

:file: `/etc/tmpfiles.d/celery.conf`

```bash
d /run/celery 0755 celery celery -
d /var/log/celery 0755 celery celery -
```

#### Example configuration

This is an example configuration for a Python project:

:file:`/etc/conf.d/celery`:

```bash
# Name of nodes to start
# here we have a single node
CELERYD_NODES="w1"
# or we could have three nodes:
#CELERYD_NODES="w1 w2 w3"

# Absolute or relative path to the 'celery' command:
CELERY_BIN="/usr/local/bin/celery"
#CELERY_BIN="/virtualenvs/def/bin/celery"

# App instance to use
# comment out this line if you don't use an app
CELERY_APP="proj"
# or fully qualified:
#CELERY_APP="proj.tasks:app"

# How to call manage.py
CELERYD_MULTI="multi"

# Extra command-line arguments to the worker
CELERYD_OPTS="--time-limit=300 --concurrency=8"

# - %n will be replaced with the first part of the nodename.
# - %I will be replaced with the current child process index
#   and is important when using the prefork pool to avoid race conditions.
CELERYD_PID_FILE="/var/run/celery/%n.pid"
CELERYD_LOG_FILE="/var/log/celery/%n%I.log"
CELERYD_LOG_LEVEL="INFO"

# you may wish to add these options for Celery Beat
CELERYBEAT_PID_FILE="/var/run/celery/beat.pid"
CELERYBEAT_LOG_FILE="/var/log/celery/beat.log"
```

### Service file: celerybeat.service

This is an example systemd file for Celery Beat:

:file:`/etc/systemd/system/celerybeat.service`:

```bash
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

Once you've put that file in :file:`/etc/systemd/system`, you should run
:command:`systemctl daemon-reload` in order that Systemd acknowledges that file.
You should also run that command each time you modify it.
Use :command:`systemctl enable celerybeat.service` if you want the celery beat
service to automatically start when (re)booting the system.

## Running the worker with superuser privileges (root)

Running the worker with superuser privileges is a very dangerous practice.
There should always be a workaround to avoid running as root. Celery may
run arbitrary code in messages serialized with pickle - this is dangerous,
especially when run as root.

By default Celery won't run workers as root. The associated error
message may not be visible in the logs but may be seen if :envvar:`C_FAKEFORK`
is used.

To force Celery to run workers as root use :envvar:`C_FORCE_ROOT`.

When running as root without :envvar:`C_FORCE_ROOT` the worker will
appear to start with *"OK"* but exit immediately after with no apparent
errors. This problem may appear when running the project in a new development
or production environment (inadvertently) as root.

## :pypi:`supervisor`

[`extra/supervisord/`](https://github.com/celery/celery/tree/main/extra/supervisord/)

## `launchd` (macOS)

[`extra/macOS`](https://github.com/celery/celery/tree/main/extra/macOS/)
