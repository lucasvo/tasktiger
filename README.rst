=========
TaskTiger
=========
.. image:: https://circleci.com/gh/closeio/tasktiger/tree/master.svg?style=svg&circle-token=a86617952aa9b4cfee784b6ac43358cd042a6672
    :target: https://circleci.com/gh/closeio/tasktiger/tree/master

*TaskTiger* is a Python task queue using Redis.


(Interested in working on projects like this? `Close.io`_ is looking for `great engineers`_ to join our team)

.. _Close.io: http://close.io
.. _great engineers: http://jobs.close.io


.. contents:: Contents

Features
--------

- Per-task fork

  TaskTiger forks a subprocess for each task, This comes with several benefits:
  Memory leaks caused by tasks are avoided since the subprocess is terminated
  when the task is finished. A hard time limit can be set for each task, after 
  which the task is killed if it hasn't completed. To ensure performance, any
  necessary Python modules can be preloaded in the parent process.

- Unique queues

  TaskTiger has the option to avoid duplicate tasks in the task queue. In some
  cases it is desirable to combine multiple similar tasks. For example, imagine
  a task that indexes objects (e.g. to make them searchable). If an object is
  already present in the task queue and hasn't been processed yet, a unique
  queue will ensure that the indexing task doesn't have to do duplicate work.
  However, if the task is already running while it's queued, the task will be
  executed another time to ensure that the indexing task always picks up the
  latest state.

- Task locks

  TaskTiger can ensure to never execute more than one instance of tasks with
  similar arguments by acquiring a lock. If a task hits a lock, it is requeued
  and scheduled for later executions after a configurable interval.

- Task retrying

  TaskTiger lets you retry exceptions (all exceptions or a list of specific
  ones) and comes with configurable retry intervals (fixed, linear,
  exponential, custom).

- Flexible queues

  Tasks can be easily queued in separate queues. Workers pick tasks from a
  randomly chosen queue and can be configured to only process specific queues,
  ensuring that all queues are processed equally. TaskTiger also supports
  subqueues which are separated by a period. For example, you can have
  per-customer queues in the form ``process_emails.CUSTOMER_ID`` and start a
  worker to process ``process_emails`` and any of its subqueues. Since tasks
  are picked from a random queue, all customers get equal treatment: If one
  customer is queueing many tasks it can't block other customers' tasks from
  being processed.

- Batch queues

  Batch queues can be used to combine multiple queued tasks into one. That way,
  your task function can process multiple sets of arguments at the same time,
  which can improve performance. The batch size is configurable.

- Scheduled tasks

  Tasks can be scheduled for execution at a specific time.

- Structured logging

  TaskTiger supports JSON-style logging via structlog, allowing more
  flexibility for tools to analyze the log. For example, you can use TaskTiger
  together with Logstash, Elasticsearch, and Kibana.

- Reliability

  TaskTiger atomically moves tasks between queue states, and will re-execute
  tasks after a timeout if a worker crashes.

- Error handling

  If an exception occurs during task execution and the task is not set up to be
  retried, TaskTiger stores the execution tracebacks in an error queue. The
  task can then be retried or deleted manually. TaskTiger can be easily
  integrated with error reporting services like Rollbar.

- Admin interface

  A simple admin interface using flask-admin exists as a separate project
  (tasktiger-admin_).

.. _tasktiger-admin: https://github.com/closeio/tasktiger-admin


Quick start
-----------

It is easy to get started with TaskTiger.

Create a file that contains the task(s).

.. code:: python

  # tasks.py
  def my_task():
      print 'Hello'

Queue the task using the ``delay`` method.

.. code:: python

  In [1]: import tasktiger, tasks
  In [2]: tiger = tasktiger.TaskTiger()
  In [3]: tiger.delay(tasks.my_task)

Run a worker.

.. code:: bash

  % tasktiger
  {"timestamp": "2015-08-27T21:00:09.135344Z", "queues": null, "pid": 69840, "event": "ready", "level": "info"}
  {"task_id": "6fa07a91642363593cddef7a9e0c70ae3480921231710aa7648b467e637baa79", "level": "debug", "timestamp": "2015-08-27T21:03:56.727051Z", "pid": 69840, "queue": "default", "child_pid": 70171, "event": "processing"}
  Hello
  {"task_id": "6fa07a91642363593cddef7a9e0c70ae3480921231710aa7648b467e637baa79", "level": "debug", "timestamp": "2015-08-27T21:03:56.732457Z", "pid": 69840, "queue": "default", "event": "done"}


Configuration
-------------

A ``TaskTiger`` object keeps track of TaskTiger's settings and is used to
decorate and queue tasks. The constructor takes the following arguments:

- ``connection``

  Redis connection object

- ``config``

  Dict with config options. Most configuration options don't need to be
  changed, and a full list can be seen within ``TaskTiger``'s ``__init__``
  method.

- ``setup_structlog``

  If set to True, sets up structured logging using ``structlog`` when
  initializing TaskTiger. This makes writing custom worker scripts easier
  since it doesn't require the user to set up ``structlog`` in advance.

Example:

.. code:: python

  import tasktiger
  from redis import Redis
  conn = redis.Redis(db=1)
  tiger = tasktiger.TaskTiger(connection=conn, config={
      'BATCH_QUEUES': { 'batch': 10 },
  })


Task decorator
--------------

TaskTiger provides a task decorator to specify task options. Note that simple
tasks don't need to be decorated. However, decorating the task allows you to
use an alternative syntax to queue the task, which is compatible with Celery:

.. code:: python

  # tasks.py

  import tasktiger
  tiger = tasktiger.TaskTiger()

  @tiger.task()
  def my_task(name, n=None):
      print 'Hello', name

.. code:: python

  In [1]: import tasks
  # The following are equivalent. However, the second syntax can only be used
  # if the task is decorated.
  In [2]: tasks.tiger.delay(my_task, args=('John',), kwargs={'n': 1})
  In [3]: tasks.my_task.delay('John', n=1)


Task options
------------

Tasks support a variety of options that can be specified either in the task
decorator, or when queueing a task. For the latter, the ``delay`` method must
be called on the ``TaskTiger`` object, and any options in the task decorator
are overridden.

.. code:: python

  @tiger.task(queue='myqueue', unique=True)
  def my_task():
      print 'Hello'

.. code:: python

  # The task will be queued in "otherqueue", even though the task decorator
  # says "myqueue".
  tiger.delay(my_task, queue='otherqueue')

When queueing a task, the task needs to be defined in a module other than the
Python file which is being executed. In other words, the task can't be in the
``__main__`` module. TaskTiger will give you back an error otherwise.

The following options are supported for ``delay``:

- ``queue``

  Name of the queue where the task will be queued.

- ``hard_timeout``

  If the task runs longer than the given number of seconds, it will be
  killed and marked as failed.

- ``unique``

  The task will only be queued if there is no similar task with the
  same function, arguments, and keyword arguments in the queue. Note
  that multiple similar tasks may still be executed at the same time
  since the task will still be inserted into the queue if another one
  is being processed.

- ``lock``

  Hold a lock while the task is being executed (with the given args and
  kwargs). If a task with similar args/kwargs is queued and tries to
  acquire the lock, it will be retried later.

- ``lock_key``

  If set, this implies lock=True and specifies the list of kwargs to
  use to construct the lock key. By default, all args and kwargs are
  serialized and hashed.

- ``when``

  Takes either a datetime (for an absolute date) or a timedelta
  (relative to now). If given, the task will be scheduled for the given
  time.

- ``retry``

  Whether to retry a task when it fails (either because of an exception
  or because of a timeout). To restrict the list of failures, use
  retry_on. Unless retry_method is given, the configured
  ``DEFAULT_RETRY_METHOD`` is used.

- ``retry_on``

  If a list is given, it implies ``retry=True``. Task will be only retried
  on the given exceptions (or its subclasses). To retry the task when a
  hard timeout occurs, use ``JobTimeoutException``.

- ``retry_method``

  If given, implies ``retry=True``. Pass either:

  - a function that takes the retry number as an argument, or,
  - a tuple ``(f, args)``, where ``f`` takes the retry number as the first
    argument, followed by the additional args.

  The function needs to return the desired retry interval in seconds,
  or raise StopRetry to stop retrying. The following built-in functions
  can be passed for common scenarios and return the appropriate tuple:

  - ``fixed(delay, max_retries)``

    Returns a method that returns the given delay or raises StopRetry
    if the number of retries exceeds max_retries.

  - ``linear(delay, increment, max_retries)``

    Like fixed, but starts off with the given delay and increments it
    by the given increment after every retry.

  - ``exponential(delay, factor, max_retries)``

    Like fixed, but starts off with the given delay and multiplies it
    by the given factor after every retry.

The following options can be only specified in the task decorator:

- ``batch``

  If set to ``True``, the task will receive a list of dicts with args and
  kwargs and can process multiple tasks of the same type at once.
  Example: ``[{"args": [1], "kwargs": {}}, {"args": [2], "kwargs": {}}]``
  Note that the list will only contain multiple items if the worker
  has set up ``BATCH_QUEUES`` for the specific queue.


Custom retrying
---------------

In some cases the task retry options may not be flexible enough. For example,
you might want to use a different retry method depending on the exception type,
or you might want to like to suppress logging an error if a task fails after
retries. In these cases, ``RetryException`` can be raised within the task
function. The following options are supported:

- ``method``

  Specify a custom retry method for this retry. If not given, the task's
  default retry method is used, or, if unspecified, the configured
  ``DEFAULT_RETRY_METHOD``. Note that the number of retries passed to the
  retry method is always the total number of times this method has been
  executed, regardless of which retry method was used.

- ``original_traceback``

  If ``RetryException`` is raised from within an except block and
  ``original_traceback`` is True, the original traceback will be logged (i.e.
  the stacktrace at the place where the caught exception was raised). False by
  default.

- ``log_error``

  If set to False and the task fails permanently, a warning will be logged
  instead of an error, and the task will be removed from Redis when it
  completes. True by default.

Example usage:

.. code:: python

  from tasktiger.exceptions import RetryException

  def my_task():
      if not ready():
          # Retry every minute up to 3 times if we're not ready. An error will
          # be logged if we're out of retries.
          raise RetryException(method=fixed(60, 3))

      try:
          some_code()
      except NetworkException:
          # Back off exponentially up to 5 times in case of a network failure.
          # Log the original traceback (as a warning) and don't log an error if
          # we still fail after 5 times.
          raise RetryException(method=exponential(60, 2, 5),
                               original_traceback=True,
                               log_error=False)


Workers
-------

The ``tasktiger`` command is used on the command line to invoke a worker. To
invoke multiple workers, multiple instances need to be started. This can be
easily done e.g. via Supervisor. The following Supervisor configuration file
can be placed in ``/etc/supervisor/tasktiger.ini`` and runs 4 TaskTiger workers
as the ``ubuntu`` user. For more information, read Supervisor's documentation.

.. code:: bash

  [program:tasktiger]
  command=/usr/local/bin/tasktiger
  process_name=%(program_name)s_%(process_num)02d
  numprocs=4
  numprocs_start=0
  priority=999
  autostart=true
  autorestart=true
  startsecs=10
  startretries=3
  exitcodes=0,2
  stopsignal=TERM
  stopwaitsecs=600
  killasgroup=false
  user=ubuntu
  redirect_stderr=false
  stdout_logfile=/var/log/tasktiger.out.log
  stdout_logfile_maxbytes=250MB
  stdout_logfile_backups=10
  stderr_logfile=/var/log/tasktiger.err.log
  stderr_logfile_maxbytes=250MB
  stderr_logfile_backups=10

Workers support the following options:

- ``-q``, ``--queues``

  If specified, only the given queue(s) are processed. Multiple queues can be
  separated by comma. Any subqueues of the given queues will be also processed.
  For example, ``-q first,second`` will process items from ``first``,
  ``second``, and subqueues such as ``first.CUSTOMER1``, ``first.CUSTOMER2``.

- ``-m``, ``--module``

  Module(s) to import when launching the worker. This improves task performance
  since the module doesn't have to be reimported every time a task is forked.
  Multiple modules can be separated by comma.

  Another way to preload modules is to set up a custom TaskTiger launch script,
  which is described below.

- ``-h``, ``--host``

  Redis server hostname (if different from ``localhost``).

- ``-p``, ``--port``

  Redis server port (if different from ``6379``).

- ``-a``, ``--password``

  Redis server password (if required).

- ``-n``, ``--db``

  Redis server database number (if different from ``0``).

In some cases it is convenient to have a custom TaskTiger launch script. For
example, your application may have a ``manage.py`` command that sets up the
environment and you may want to launch TaskTiger workers using that script. To
do that, you can use the ``run_worker_with_args`` method, which launches a
TaskTiger worker and parses any command line arguments. Here is an example:

.. code:: python

  import sys
  from tasktiger import TaskTiger

  try:
      command = sys.argv[1]
  except IndexError:
      command = None

  if command == 'tasktiger':
      tiger = TaskTiger(setup_structlog=True)
      # Strip the "tasktiger" arg when running via manage, so we can run e.g.
      # ./manage.py tasktiger --help
      tiger.run_worker_with_args(sys.argv[2:])
      sys.exit(0)

Note that if you're using ``flask-script``, you will still need to manually
evaluate ``sys.argv`` to ensure proper argument parsing, instead of using a
``flask-script`` command.


Inspect, requeue and delete tasks
---------------------------------

TaskTiger does not currently come with an admin interface, but provides access
to the ``Task`` class which lets you inspect queues and requeue and delete
tasks.

Each queue can have tasks in the following states:

- ``queued``: Tasks that are queued and waiting to be picked up by the workers.
- ``active``: Tasks that are currently being processed by the workers.
- ``scheduled``: Tasks that are scheduled for later execution.
- ``error``: Tasks that failed with an error.

To get a list of all tasks for a given queue and state, use
``Task.tasks_from_queue``. The method gives you back a tuple containing the
total number of tasks in the queue (useful if the tasks are truncated) and a
list of tasks in the queue, latest first. Using the ``skip`` and ``limit``
keyword arguments, you can fetch arbitrary slices of the queue. If you know the
task ID, you can fetch a given task using ``Task.from_id``. Both methods let
you load tracebacks from failed task executions using the ``load_executions``
keyword argument, which accepts an integer indicating how many executions
should be loaded.

The ``Task`` object has the following properties:

- ``id``: The task ID.

- ``data``: The raw data as a dict from Redis.

- ``executions``: A list of failed task executions (as dicts). An execution
  dict contains the processing time in ``time_started`` and ``time_failed``,
  the worker host in ``host``, the exception name in ``exception_name`` and
  the full traceback in ``traceback``.

- ``func``, ``args``, ``kwargs``: The serialized function name with all of its
  arguments.

The ``Task`` object has the following methods. Note that these methods only
work for tasks that are in the error queue.

- ``retry``: Requeue the task for execution.

- ``delete``: Remove the task from the error queue.

Example:

.. code:: python

  from tasktiger import TaskTiger
  from tasktiger.task import Task

  QUEUE_NAME = 'default'
  TASK_STATE = 'error'
  TASK_ID = '6fa07a91642363593cddef7a9e0c70ae3480921231710aa7648b467e637baa79'

  tiger = TaskTiger()

  n_total, tasks = Task.tasks_from_queue(tiger, QUEUE_NAME, TASK_STATE)

  for task in tasks:
      print task.id, task.func

  task = Task.from_id(tiger, QUEUE_NAME, TASK_STATE, TASK_ID)
  task.retry()


Rollbar error handling
----------------------

TaskTiger comes with Rollbar integration for error handling. When a task errors
out, it can be logged to Rollbar, grouped by queue, task function name and
exception type. To enable logging, initialize rollbar with the
``StructlogRollbarHandler`` provided in the ``tasktiger.rollbar`` module. The
handler takes a string as an argument which is used to prefix all the messages
reported to Rollbar. Here is a custom worker launch script:

.. code:: python

  import logging
  import rollbar
  import sys
  from tasktiger import TaskTiger
  from tasktiger.rollbar import StructlogRollbarHandler

  tiger = TaskTiger(setup_structlog=True)

  rollbar.init(ROLLBAR_API_KEY, APPLICATION_ENVIRONMENT,
               allow_logging_basic_config=False)
  rollbar_handler = StructlogRollbarHandler('TaskTiger')
  rollbar_handler.setLevel(logging.ERROR)
  tiger.log.addHandler(rollbar_handler)

  tiger.run_worker_with_args(sys.argv[1:])
