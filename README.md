# First steps with Django

## Using Celery with Django

---
**NOTE**

Previous versions of Celery required a separate library to work with Django, but since 3.1 this is no longer the case.
    Django is supported out of the box now so this document only contains a basic way to integrate Celery and Django.
    You'll use the same API as non-Django users so you're recommended to read the :ref:`first-steps` tutorial first and
    come back to this tutorial. When you have a working example you can continue to the :ref:`next-steps` guide.
---

!!! note ""

    Celery 4.0 supports Django 1.8 and newer versions. Please use Celery 3.1
    for versions older than Django 1.8.

To use Celery with your Django project you must first define an instance of the Celery library (called an "app")

If you have a modern Django project layout like::

```
- proj/
  - manage.py
  - proj/
    - __init__.py
    - settings.py
    - urls.py
```

then the recommended way is to create a new `proj/proj/celery.py` module that defines the Celery instance:

`proj/proj/celery.py`

```python
from __future__ import absolute_import, unicode_literals
import os
from celery import Celery

# set the default Django settings module for the 'celery' program.
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'proj.settings')

app = Celery('proj')

# Using a string here means the worker don't have to serialize
# the configuration object to child processes.
# - namespace='CELERY' means all celery-related configuration keys
#   should have a `CELERY_` prefix.
app.config_from_object('django.conf:settings', namespace='CELERY')

# Load task modules from all registered Django app configs.
app.autodiscover_tasks()


@app.task(bind=True)
def debug_task(self):
    print('Request: {0!r}'.format(self.request))
```

Then you need to import this app in your `proj/proj/__init__.py` module. This ensures that the app is loaded when Django
starts so that the `@shared_task` decorator (mentioned later) will use it:

`proj/proj/__init__.py`:

```python
from __future__ import absolute_import, unicode_literals

# This will make sure the app is always imported when
# Django starts so that shared_task will use this app.
from .celery import app as celery_app

__all__ = ['celery_app']
```

Note that this example project layout is suitable for larger projects, for simple projects you may use a single contained
module that defines both the app and tasks, like in the :ref:`tut-celery` tutorial.

Let's break down what happens in the first module, first we import absolute imports from the future, so that our
`celery.py` module won't clash with the library:

```python
from __future__ import absolute_import
```

Then we set the default [`DJANGO_SETTINGS_MODULE`](http://django.readthedocs.io/en/latest/topics/settings.html#envvar-DJANGO_SETTINGS_MODULE)
environment variable for the :program:`celery` command-line program:

```python
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'proj.settings')
```

You don't need this line, but it saves you from always passing in the
settings module to the ``celery`` program. It must always come before
creating the app instances, as is what we do next:

```python
app = Celery('proj')
```

This is our instance of the library, you can have many instances but there's probably no reason for that when using Django.

We also add the Django settings module as a configuration source for Celery. This means that you don't have to use multiple
configuration files, and instead configure Celery directly from the Django settings; but you can also separate them if wanted.

The uppercase name-space means that all Celery configuration options must be specified in uppercase instead of lowercase,
and start with `CELERY_`, so for example the [`task_always_eager`](http://docs.celeryproject.org/en/latest/userguide/configuration.html#std:setting-task_always_eager)
setting becomes `CELERY_TASK_ALWAYS_EAGER`, and the [`broker_url`](http://docs.celeryproject.org/en/latest/userguide/configuration.html#std:setting-broker_url)
setting becomes `CELERY_BROKER_URL`.

You can pass the object directly here, but using a string is better since then the worker doesn't have to serialize the object.

```python
app.config_from_object('django.conf:settings', namespace='CELERY')
```

Next, a common practice for reusable apps is to define all tasks in a separate `tasks.py` module, and Celery does have
a way to auto-discover these modules:

```python
app.autodiscover_tasks()
```

With the line above Celery will automatically discover tasks from all
of your installed apps, following the ``tasks.py`` convention::

```
- app1/
    - tasks.py
    - models.py
- app2/
    - tasks.py
    - models.py
```

This way you don't have to manually add the individual modules to the [`CELERY_IMPORTS <imports>`](http://docs.celeryproject.org/en/latest/userguide/configuration.html#std:setting-imports)
setting.

Finally, the `debug_task` example is a task that dumps its own request information. This is using the new `bind=True`
task option introduced in Celery 3.1 to easily refer to the current task instance.

### Using the ``@shared_task`` decorator

The tasks you write will probably live in reusable apps, and reusable apps cannot depend on the project itself, so you
also cannot import your app instance directly.

The `@shared_task` decorator lets you create tasks without having any concrete app instance:

`demoapp/tasks.py`:

```python
# Create your tasks here
from __future__ import absolute_import, unicode_literals
from celery import shared_task


@shared_task
def add(x, y):
    return x + y


@shared_task
def mul(x, y):
    return x * y


@shared_task
def xsum(numbers):
    return sum(numbers)
```

> ### See also:
> You can find the full source code for the Django example project at: [https://github.com/celery/celery/tree/master/examples/django/]()



> ### Relative Imports
> You have to be consistent in how you import the task module. For example, if you have `project.app` in `INSTALLED_APPS`,
then you must also import the tasks `from project.app` or else the names of the tasks will end up being different.
> See [Automatic naming and relative imports](http://docs.celeryproject.org/en/latest/userguide/tasks.html#task-naming-relative-imports)

## Extensions

### `django-celery-results` - Using the Django ORM/Cache as a result backend

The [django-celery-results](https://pypi.python.org/pypi/django_celery_results) extension provides result backends using
either the Django ORM, or the Django Cache framework.

To use this with your project you need to follow these steps:

1. Install the [django-celery-results](https://pypi.python.org/pypi/django_celery_results) library:

```bash
$ pip install django-celery-results
```

2. Add `django_celery_results` to `INSTALLED_APPS` in your Django project's `settings.py`:

```python
INSTALLED_APPS = (
    ...,
    'django_celery_results',
)
```

Note that there is no dash in the module name, only underscores.

3. Create the Celery database tables by performing a database migrations:

```bash
$ python manage.py migrate django_celery_results
```

4. Configure Celery to use the [django-celery-results](https://pypi.python.org/pypi/django_celery_results) backend.

Assuming you are using Django's `settings.py` to also configure Celery, add the following settings:

```python
CELERY_RESULT_BACKEND = 'django-db'
```

For the cache backend you can use:

```python
CELERY_RESULT_BACKEND = 'django-cache'
```

### `django-celery-beat` - Database-backed Periodic Tasks with Admin interface.

See [Using custom scheduler classes](http://docs.celeryproject.org/en/latest/userguide/periodic-tasks.html#beat-custom-schedulers)
for more information.

## Starting the worker process

In a production environment you'll want to run the worker in the background as a daemon - see
[Daemonization](http://docs.celeryproject.org/en/latest/userguide/daemonizing.html#daemonizing) - but for testing and
development it is useful to be able to start a worker instance by using the `celery worker` manage command, much as
you'd use Django's `manage.py runserver`:

```bash
$ celery -A proj worker -l info
```

For a complete listing of the command-line options available, use the help command:

```bash
$ celery help
```

## Where to go from here

If you want to learn more you should continue to the [Next Steps](http://docs.celeryproject.org/en/latest/getting-started/next-steps.html#next-steps)
tutorial, and after that you can study the [User Guide](http://docs.celeryproject.org/en/latest/userguide/index.html#guide).
