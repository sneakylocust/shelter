Shelter
=======

Shelter is a *Python's Tornado* wrapper which provides classes and helpers
for creation new application skeleton, writing management commands (like a
*Django*), service processes and request handlers. It was tested with
*Python 2.7* and *Python 3.4* or higher and *Tornado 3.2* or higher.

Instalation
-----------

::

    cd shelter/
    python setup.py install

After instalation **shelter-admin** command is available. For help type:

::

    shelter-admin -h

The most important argument is ``-s/--settings``, which joins Shelter library
and your application. Format is Python's module path, eg. `myapp.settings`.
Second option how to handle `settings` module is ``SHELTER_SETTINGS_MODULE``
environment variable. If both are handled, command line argument has higher
priority than environment variable.

::

    shelter-admin -s myapp.settings

or

::

    SHELTER_SETTINGS_MODULE=myapp.settings shelter-admin

Usage
------

::

    shelter-admin startproject myproject

Skeleton of the new application will be created in current working directory.
Project name has the same rules as Python's module name. Entry point into new
application is a script **manage.py**.

::

    cd myproject/
    ./manage.py -h
    ./manage.py devserver

`settings.py` is included in the new skeleton. See comments in file how to
define interfaces, management commands, service processes, ...

Management commands which provides Shelter library:

+ **devserver** runs development HTTP server, which autoreloads application
  when source files are changes. Server is run only in one process, service
  processes are run in threads.
+ **runserver** runs production HTTP, multi-process server. Number of the
  processes are detected according to ``INTERFACES`` setting in the
  ``settings`` module. Service processes are run in separated processes.
  Parent process checks child processes and when child process crashes,
  it is run again. Maximum amount of the crashes are 100, then application
  will exit.
+ **shell** runs interactive Python's shell. First it tries to run *IPython*,
  then standard *Python* shell. Service processes are run in threads.
+ **showconfig** shows effective configuration.
+ **startproject** will generate new apllication skeleton.

Config class
------------

Library provides base configuration class ``shelter.core.config.Config``
which holds all configuration. Public attributes are **settings** which
is ``settings`` module of the application and **args_parser** which is
instance of the ``argparse.ArgumentParser`` from *Python's standard library*.

You can override this class in the `settings` module::

    CONFIG_CLASS = 'myapp.core.config.AppConfig'

Your own `AppConfig` class can contain additional *properties* with
application's settings, e.g. database connection arguments. Way how the value
is found is only on you - either only in **settings** or **args_parser** or
in both. You can define additional arguments of the command line.

::

    import time

    from shelter.core.config import Config, argument

    class AppConfig(Config):

        arguments = (
            argument(
                '-k', '--secret-key',
                dest='secret_key', action='store',
                type=str, default='',
                help='configuration file'
            ),
        )

        Database = collections.namedtuple('Database', ['host', 'db'])

        def initialize(self):
            # initialize() is called after instance is created. If you want
            # add some instance attributes, use this method instead of
            # override __init__().
            self._server_started_time = time.time()

        def get_config_items(self):
            # Override get_config_items() method if you want to add
            # your options into showconfig management command.
            base_items = super(AppConfig, self).get_config_items()
            app_items = (
                ('secret_key', self.secret_key),
                ('database', self.database),
            )
            return base_items + app_items

        @property
        def secret_key(self):
            # If command-line argument -k/--secret-key exists, it will
            # override settings.SECRET_KEY value.
            return self.args_parser.secret_key or self.settings.SECRET_KEY

        @property
        def database(self):
            return self.Database(
                db=self.settings.DATABASE_NAME,
                host=self.settings.DATABASE_HOST,
                passwd=getattr(self.settings, DATABASE_PASSWORD, '')
            )

Context class
-------------

In all handlers, management commands and service processes is available
instance of the ``shelter.core.context.Context`` which holds data and
classes instance for your appllication. Bundled class ``Context`` contains
only one property **config** with ``Config`` instance (see previous
chapter).

You can define own class in ``settings`` module::

    CONTEXT_CLASS = 'myapp.core.context.Context'

Overrided ``Context`` can contain additional *properties*, e.g. database
connection pool.

**It is necesary to initialize shared sources (sockets, open files, ...)
lazy!** The reason is that subprocesses (Tornado HTTP workers, service
processes) have to get uninitialized ``Context``, because forked resources
can cause a lot of nights without dreams... **Also it is necessary to known
that Context is shared among coroutines!** So you are responsible for
locking shared resources (be carreful, it is blocking operation) or use
another mechanism, e.g. database connection pool.

::

    class Context(shelter.core.context.Context):

        def initialize(self):
            self._database = None

        @property
        def database(self):
            if self._database is None:
                self._database = ConnectionPool(
                    self.config.database.host,
                    self.config.database.db)
            return self._database

Hooks
-----

You can define several hooks in the ``settings`` module - when application
is launched and on **SIGUSR1** and **SIGUSR2** signals.

::

    INIT_HANDLER = 'myapp.core.app.init_handler'
    SIGUSR1_HANDLER = 'myapp.core.app.sigusr1_handler'
    SIGUSR2_HANDLER = 'myapp.core.app.sigusr2_handler'

Handler is common *Python's* function which takes only one argument
*context* with ``Context`` instance (see previous chapter).

::

    def init_handler(context):
        do_something(context.config)

+ **INIT_HANDLER** is called during the application starts, before workers
  or service processes are run.
+ **SIGUSR1_HANDLER** is called on **SIGUSR1** signal. When signal receives
  worker/child process, it is processed only in this process. When signal
  receives main/parent process, signal is propagated into all workers.
+ **SIGUSR2_HANDLER** is called on **SIGUSR2** signal. Signal is processed
  only in process which received signal. It is not propagated anywhere.

Service processes
-----------------

Service process are tasks which are repeatedly launched in adjusted interval,
e.g. warms cache data before they expire. Library provides base class
``shelter.core.process.BaseProcess``. For new service process
you must inherit ``BaseProcess``, adjust ``interval`` attribute and override
``loop()`` method.

::

    from shelter.core.processes import BaseProcess

    class WarmCache(BaseProcess)

        interval = 30.0

        def initialize(self):
            self.db_conn = self.context.db.conn_pool
            self.cache = self.context.cache

        def loop(self):
            self.logger.info("Warn cached data")
            with self.db_conn.get() as db:
                self.cache.set('key', db.get_data(), timeout=60)

+ **interval** is a time in seconds. After this time ``loop()`` method is
  repeatedly called.

Service process has to be registered in the ``settings`` module.

::

    SERVICE_PROCESSES = (
        ('myapp.processes.WarmCache', True, 15.0),
    )

Each service process definition is list/tuple in format
``('path.to.ClassName', wait_unless_ready, timeout)``. If *wait_unless_ready*
is ``True``, wait maximum *timeout* seconds unless process is successfully
started, otherwise raise ``shelter.core.exceptions.ProcessError`` exception.

Management commands
-------------------

Class ``shelter.core.commands.BaseCommand`` is an ancestor for user
defined managemend commands, e.g. export/import database data. For new
management command you must inherit ``BaseCommand`` and override ``command()``
method and/or ``initialize()`` method.

::

    import sys

    from gettext import gettext as _

    from shelter.core.commands import BaseCommand, argument

    class Export(BaseCommand)

        name = 'export'
        help = 'export data from database'
        arguments = (
            argument(
                '-o', dest=output_file, type=str, default='-',
                help=_('output filename')),
        )

        def initialize(self):
            filename = self.conntext.config.args_parser.output_file
            if filename == '-':
                self.output_file = sys.stdout
            else:
                self.output_file = open(filename, 'wt')

        def command(self):
            self.logger.info("Exporting data")
            with self.context.db.get_connection_from_pool() as db:
                data = db.get_data()
            self.output_file.write(data)
            self.output_file.flush()

+ **name** is a name of the management command. This name you type into
  command line, e.g. ``./manage.py export``.
+ **help** is a short description of the management command. This help is
  printed onto console when you type ``./manage.py command -h``.
+ **arguments** are arguments of the command line parser. ``argument()``
  function has the same meaning as ``ArgumentParser.add_argument()``
  from *Python's standard library*.
+ **service_processes_start** If it is ``True``, service processes will be
  launched on background. Default is do not launch any service processes.
  **It is not public API, do not use this attribute unless you really know
  what you are doing**!
+ **service_processes_in_thread** If it is ``True``, launch service
  processes in threads, else as a separated processes. **It is not public
  API, do not use this attribute unless you really know what you are doing**!
+ **settings_required** If it is ``True``, `settings` module will not be
  required. **It is not public API, do not use this attribute unless you
  really know what you are doing**!

Management command has to be registered in the ``settings`` module.

::

    MANAGEMENT_COMMANDS = (
        'myapp.commands.Export',
    )

Interfaces
----------

*Tornado's HTTP server* can be run in multiple instances. Interface are
defined in ``settings`` module.

::

    INTERFACES = {
        'default': {
            # IP/hostname (not required) and port where the interface
            # listen to requests
            'LISTEN': ':8000',

            # Amount of the server processes if application is run
            # using runserver command. Positive integer, 0 will
            # detect amount of the CPUs
            'PROCESSES': 0,

            # Path in format 'path.to.module.variable_name' where
            # urls patterns are defined
            'URLS': 'myapp.urls.urls_default',
        },
    }

URL path to HTTP handler routing
--------------------------------

It is the same as in *Python's Tornado* application.

::

    from tornado.web import URLSpec

    from myapp.handlers import HomepageHandler, AboutHandler

    urls_default = (
        URLSpec(r'/', HomepageHandler),
        URLSpec(r'/about/', AboutHandler),
    )

Tuple/list **urls_default** is handled into relevant interface in the
``settings`` module, see previous chapter.

HTTP handler is a subclass of the ``shelter.core.web.BaseRequestHandler``
which enhances ``tornado.web.RequestHandler``. Provides additional instance
attributes/properties **logger**, **context** and **interface**.

+ **logger** is an instance of the ``logging.Logger`` from *Python's standard
  library*. Logger name is derived from handlers's name, e.g
  ``myapp.handlers.HomepageHandler``.
+ **context** is an instance of the ``Context``, see *Context* paragraph.
+ **interface** is a namedtuple with informations about current interface.
  Named attributes are **name**, **host**, **port**, **processes** and
  **urls**.

::

    from shelter.core.web import BaseRequestHandler

    class DummyHandler(BaseRequestHandler):

        def get(self):
            self.write(
                "Interface '%s' works!\n" % self.interface.name)
            self.set_header(
                "Content-Type", 'text/plain; charset=UTF-8')

Logging
-------

Standard *Python's logging* is used. ``Config.configure_logging()`` method
is responsible for setting the logging. Default ``Config`` class reads
logging's configuration from ``settings`` module::

    LOGGING = {
        'version': 1,
        'disable_existing_loggers': False,
        'handlers': {
            'console': {
                'class': 'logging.StreamHandler',
                'level': 'INFO',
                'formatter': 'default',
            },
        },
        'root': {
            'handlers': ['console'],
            'level': 'INFO',
        },
    }

Contrib
-------

shelter.contrib.config.iniconfig.IniConfig
``````````````````````````````````````````

Descendant of the ``shelter.core.config.Config``, provides **INI** files
configuration. Adds additional public attribute **config_parser** which is
instance of the ``RawConfigParser`` from *Python's standard library*.
Interfaces and application's name can be overrided in configuration file,
*Python's logging* must be defined.

Configuration file is specified either by ``SHELTER_CONFIG_FILENAME``
environment variable or ``-f/--config-file`` command line argument. First,
main configuration file is read. Then all configuration files from
``file.conf.d`` subdirectory are read in alphabetical order. E.g. if
``-f conf/myapp.conf`` is handled, first ``conf/myapp.conf`` file is read
and then all ``conf/myapp.conf.d/*.conf`` files. Value in later
configuration file overrides previous defined value.

::

    [application]
    name = MyApp

    [interface_http]
    Listen=:4444
    Processes=8
    Urls=tests.urls1.urls_http

    [formatters]
    keys=default

    [formatter_default]
    class=logging.Formatter
    format=%(asctime)s %(name)s %(levelname)s: %(message)s

    [handlers]
    keys=console

    [handler_console]
    class=logging.StreamHandler
    args=()
    level=NOTSET

    [loggers]
    keys=root

    [logger_root]
    level=INFO
    handlers=console

License
-------

3-clause BSD
