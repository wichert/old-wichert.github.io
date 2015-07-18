---
title: Task queues
description: |
  When writing a web application or REST backend I occasionally run into a
  common problem: sometimes I need to do things that will take a while: sending a
  push notification to mobile apps, generating emails or doing an expensive
  calculation. Doing this immediately would result in very long response times,
  which is not acceptable. That means I need a way to offload those to something
e  lse. This is where task queues come in: they allow you to submit a task for
  out-of-band processing. There are multiple tools that do this, but all of them
  suffer from one or more design flaws that makek them harder to use in non-trivial
  systems. This document tries to address a few requirements and looks at how
  existing frameworks try to address them.
---
When writing a web application or REST backend I occasionally run into a
common problem: sometimes I need to do things that will take a while: sending a
push notification to mobile apps, generating emails or doing an expensive
calculation. Doing this immediately would result in very long response times,
which is not acceptable. That means I need a way to offload those to something
else. This is where task queues come in: they allow you to submit a task for
out-of-band processing. There are multiple tools that do this, but all of them
suffer from one or more design flaws that makek them harder to use in non-trivial
systems. This document tries to address a few requirements and looks at how
existing frameworks try to address them.

The examples use Pyramid, but the problems are generic and apply to all
frameworks.


Global configuration
--------------------

A common fallacy is requiring the configuration to be defined globally before
you can define a task. Here is an example from
[Celery](http://www.celeryproject.org/):

```python
from celery import Celery

app = Celery('tasks', broker='amqp://guest@localhost//')

@app.task
def add(x, y):
    return x + y

add.delay(4, 4)
```

or the [rq](http://python-rq.org/) version (note that this style is optional with rq)

```python
from rq.decorators import job

@job('low', connection=my_redis_conn, timeout=5)
def add(x, y):
    return x + y
```

This pattern makes it almost impossible to make your application configurable:
it requires your connection details to be set on a global before you can import
any code that defines a task, which will quickly lead to complex import depenedencies
and can be impossible with some frameworks. A better approach is be to decouple
configuration from the task definition. For example (from rq again):

```
def add(x, y):
    return x + y

redis_conn = Redis()
q = Queue(connection=redis_conn)  # no args implies the default queue
job = q.enqueue(add, 4, 4)
```


Transparency
------------

When you write an expensive function that you want to delay, you also want
to be able to call it immediately in some situations, for example when you
call it from another function that is itself delayed. Building on our example
we might want to call ``add`` directly from a new ``add_and_multiply`` function.


```python
def add(x, y):
    reeturn x + y


def add_and_multiple(x, y):
    return (add(x, y), x ** y))  # Do not delay this call to add
```

There should also not be any requirements on function naming. Celery has
a problem here: it does not allow you to define two tasks with the same
name in different modules. If you do this:

```python
# view1.py
@task()
def send_mail():
    pass

# view2.py
@task()
def send_mail()
    pass
```

If you call ``view2.send_mail`` Celery will happily run ``view1.send_mail`
instead without telling you.


Task context
------------

Tasks commonly expect to run within a configured environment, similar to how
tests often require a fixture: a working database connection, a transaction
manager, application configuration available, etc. There are three types
of context to distinguish:

* Global state you can load once, for example application configuration.
* Per-process state that has to be initialised for every new process. Things
  like database connections fall in this category. For an application using
  Pyramid you would call
  [pyramid.paster.bootstrap](http://docs.pylonsproject.org/projects/pyramid/en/latest/api/paster.html#pyramid.paster.bootstrap)
  to set this up.
* Per-task state. Common examples of this are running each task in a separate
  transaction and managing thread-local variables.

For task context you may need more control to be able to cleanly handle
exceptions or job teardown. As an example to run a task in a transaction you
may want to use something like this:

```python
import transaction

def transaction_state(func):
    with transaction.manager as tx:
        result = func()
	tx.commit()
	return result

worker.add_task_state_handler(transaction_state)
```

A concept like [Pyramid's
tweens](http://docs.pylonsproject.org/projects/pyramid/en/latest/narr/hooks.html#registering-tweens)
may be useful here.

Celery seems to be the only framework which tries to address this with its
[signals](http://docs.celeryproject.org/en/latest/userguide/signals.html)
mechanism.


Parameter handling
------------------

When calling an expensive function that must be run out-of-process I do not
want to have to worry about what parameters I can safely pass to it. For
example if I am using [SQLAlchemy](http://sqlalchemy.org) I want to be able to
just pass an ORM instance to a function. The system should automatically detect
that so it can flush any state out to the SQL server or abort if the object is
dirty, and before running my function in a worker process merge the instances
into the current session. This is some pseudo-code for a Pyramid view to register
new users:

```python
def send_welcome_mail(user):
    """Generate and send a welcome email to a new user.
    """


@view_config('register', renderer='welcome.pt')
def register_user(request):
    user = User(email=request.params['email'])
    DBSession.add(user)
    request.registry['task-queue'].submit(send_welcome_mail, user)
    return {'user': user}
```

This is also useful to allow passing requests. A standard request has many things
such as open file handles and a copy of the request body that you can now safely
transfer to a task queue, but you do want to transfer any data over that is needed
to generate URLs with the right scheme, hostname and port. To make this transparent
that requires stripping a request down before queueing the task, and recreating
it before running the task.

```python
def send_push_message(request, device_token):
    site_url = request.route_url('news')
    message = Message([device_token],
        alert='Our website has been updated. Please visit %s' % site_url)
    apn_server.submit(message)


@view_config('publish', renderer='publish.pt')
def publish_page(request):
    ....
    request.registry['task-queue'].submit(send_push_message, user.token)
    return {}
```

[Malthe Borch](https://www.maltheborch.com) pointed out using a ``__reduce__``
method can help here. He uses [this
snippet](https://gist.github.com/malthe/b03cad86c4f9c4382045) to handle
pickling of SQLAlchemy instances. None of the existing frameworks try to address this.


Transaction integration
-----------------------

When writing a function to register new users I submit a task for later processing
to generate a welcome email for the new user. But what if I hit a critical error
after submitting that task that causes the new user not be created in my database?
In that situations the task needs to be aborted before it has a chance to run. This
requires integration of the task queue with the transaction manager.

The example below has a critical error which causes the transaction to be aborted.
In that case the transaction will be aborted and the user will be shown an error,
so it would be very confusing if the welcome email was still send.


```python
@view_config('register', renderer='welcome.pt')
def register_user(request):
    user = User(email=request.params['email'])
    DBSession.add(user)
    request.registry['task-queue'].submit(send_welcome_mail, user)
    return {'user': usr}  # Code error: usr is undefined
```

Sometimes you may need to return a task id to the user so he can come back later
to fetch a result. This requires two-phase commit support from the task queue so
it can allocate a task id before submitting the task.

```python
@view_config('rest-call', renderer='json')
def rest_call(request):
    # Schedule an expensive function
    task = request.registry['task-queue'].submit(expensive_function)
    # Tell client to check the task status in 10 seconds.
    request.response.status_int = 202
    request.headers['Location'[ = request.route_url('task-status', task_id=task.id)
    return {'task': task.id, 'delay': 10}
```

None of the existing frameworks try to address this.
