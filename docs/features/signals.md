# Signals

## 1. Basics

Django's signal dispatcher allows _senders_ to notify _receivers_ (or handlers) that an action has occurred. This is helpful for decoupling code and when you have different pieces of code that are interested in the same event. Signals are like event-based programming. You can hook up callback functions that get executed when specific events happen.

Some of the most common built-in signals in Django are:

* `django.db.models.signals.pre_save` and `django.db.models.signals.post_save` - These signals are triggered before or after a model instance `save` method is called.
* `django.db.models.signals.pre_delete` and `django.db.models.signals.post_delete` - These signals are triggered before or after a model instance's `delete` method is called and before or after a queryset's `delete` method is called.
* `django.core.signals.request_started` and `django.core.signals.request_finished` - These signals are triggered when Django starts or finishes an HTTP request.
* `django.contrib.auth.signals.user_logged_in` and `django.contrib.auth.signals.user_logged_out` - These signals are triggered after a user successfully logs in or out of the application.

The most basic use of signals is when you connect your callable with a signal to be called when that signal is triggered.

For example, lets say we want to log every request that comes into out applications. We could create a function that would do that and connect it with the `request_started` signal. Assume that we have an application called `todos`:

```python
# todos/tasks.py

def log_request(sender, **kwargs):
    print('New Request!')
```

We then need to connect the function with the signal we want. This can be done in the default configuration class's `ready` method of the `todos` application's `app.py` file.

```python
# todos/app.py
from django.apps import AppConfig

from django.core.signals import request_started

from .tasks import log_request


class TodosConfig(AppConfig):
    name = 'todos'

    def ready(self):
        request_started.connect(log_request)
```
That is all. Now every time a request comes into the application, the message "New Request!" will be logged in stdout.

The `request_started` signals sends in the `kwargs` dict only one argument, `environ`, we could enhance our signal handler function to log something a little more helpful.

```python
# todos/tasks.py

def log_request(sender, environ, **kwargs):
    method = environ['REQUEST_METHOD']
    host = environ['HTTP_HOST']
    path = environ['PATH_INFO']
    query = environ['QUERY_STRING']
    query = '?' + query if query else ''
    print('New Request -> {method} {host}{path}{query}'.format(
        method=method,
        host=host,
        path=path,
        query=query,
    ))
```
Now a more helpful message will be logged. The new message will look something like this: `New Request -> GET 127.0.0.1:8000/`. Note also that it is common/good practice to identify the arguments for the signal you are writing the handler for that will be used. In our code above we do this by adding `environ` specifically to the arguments list.

While the `request_started` signal handler we created might not be very useful there are more useful signals that can be used. The `django.db.models.signals.post_save` signal if useful for hooking in things that need to be done when a specific model is saved.

Assume, for example, that our `todos` application has a user profile model (`UserProfile`) that it wants to make sure is created for every user created in the system. One way to do this would be to connect a handler to the `post_save` signal for the `User` model in `django.contrib.auth`.

```python
# todos/tasks.py
from .models import UserProfile

...

def save_or_create_user_profile(sender, instance, created, **kwargs):
    if created:
        UserProfile.objects.create(user=instance)
    else:
        instance.user_profile.save()
```

Then in `todos/app.py` we connect the signal handler to the signal.

```python
# todos/app.py
...
from django.contrib.auth.models import User
from django.db.models.signals import post_save

from .tasks import log_request, save_or_create_user_profile


class TodosConfig(AppConfig):
    ...

    def ready(self):
        ...
        post_save.connect(save_or_create_user_profile, sender=User)
```

Note that this time we pass the `sender` argument when we connect our function to the `post_save` signal. This means that our `save_or_create_user_profile` function will only be called if the sender of the `post_save` signal is the `User` model. This also means that we can hookup to the `post_save` signal with many different models executing different code to do different things.

Finally, it is important to understand that signals are synchronous or blocking. This means that all signal handlers will have to finish executing before the code can continue after a signal has been triggered. If you have 10 signal handlers connected to the `request_started` signal, all 10 handlers would need to finish before Django even got to executing the middleware portion of the request. Keep this in mind when using signals and try not to use too many or do too much in them.

### Creating and triggering custom signals

Django also allows you to create and trigger your own signals. This can be especially useful when creating a reusable package and you want developers down the road to be able to hook in bit of functionality to your existing code in a proven way. It can also be useful within your larger Django project. Lets look at an example of this.

```python
# todo/tasks.py
import django.dispatch

todo_complete = django.dispatch.Signal(providing_args=['todo'])
```

In order to create a new signal it is as simple as instantiating a new `Signal` instance and telling it what additional arguments you will be providing handlers.

Then to send a signal:

```python
# todo/models.py
...
from .signals import todo_complete
...
class Todo(models.Model):
    description = models.CharField(max_length=255)
    done = models.BooleanField(default=False)

    def mark_complete(self):
        if not self.done:
            self.done = True
            self.save()
            todo_complete.send(sender=self.__class__, todo=self)
```
Note the call to `todo_complete.send`. This is calling the `send` method of the custom signal instance. This setup allows other developers, or yourself in other areas of your code, to hook into the event (signal) of completing a todo item and attach any functionality they would like to that event.

## 2. Deep Dive

### 2.1 Code Walk-through

#### `django.core.signals`

Django's signals implementation starts in the `django.core.signals` module ([code](https://github.com/django/django/blob/master/django/core/signals.py)). The first thing you may notice is that that module is very small (6 lines of code). All of the signals implementation is in the `django.dispatch` package ([code](https://github.com/django/django/blob/master/django/dispatch/dispatcher.py)).

What is in the signals module are the core signals that Django provides: `request_started`, `request_finished`, `got_request_exception`, `setting_changed`. These provide good examples of what you need to do in order to create your own signals.

```python
# django/core/signals.py
from django.dispatch import Signal

request_started = Signal(providing_args=["environ"])
...
```

Note that the `request_started` signal that Django provides is simply an instance of the `Signal` class provided by `django.dispatch`.

#### `django.dispatch.dispatcher.Signal`

The `Signal` class ([implementation](https://github.com/django/django/blob/master/django/dispatch/dispatcher.py#L19)) is found in the `django.dispatch.dispatcher` module (but can be imported from `django.dispatch`).

We are going to take a closer look at three methods found in the `Signal` class: `connect`, `send`, and `send_robust`.

The `connect` method is called by developer code when they want to connect a callable to the signal. The `connect` method takes only one required arguments, the handler that should be called when the signals `send` method is called.

Note that signal handlers are referred to as `receivers` in the code.

```python
# django/dispatch/dispatcher.py
...
class Signal:
    ...
    def connect(self, receiver, sender=None, weak=True, dispatch_uid=None):
        ...
        if dispatch_uid:
            lookup_key = (dispatch_uid, _make_id(sender))
        else:
            lookup_key = (_make_id(receiver), _make_id(sender))
        ...
        with self.lock:
            ...
            for r_key, _ in self.receivers:
                if r_key == lookup_key:
                    break
            else:
                self.receivers.append((lookup_key, receiver))
            ...
    ...
...
```
All this method really accomplishes is that is adds the receiver function to the list of receivers held by the signal (instance of Signal)

The `connect` method above has been simplified from the actual implementation to help us understand what `connect` is doing. First a `lookup_key` is generated for the receiver. This allows the signal to identify the individual receivers TODO.

Then `self.lock` is used in a context manager to lock the signal for adding the receiver to make sure the `self.receiver` list is only modified by one process at a time.

Inside the lock context manager block, the list of receivers is checked to make sure we don't add a duplicate (based on the `lookup_key`) and finally added if there isn't one already.

So now our receiver had been added to the signals `receiver` list. This list is then used by the `send` method.

```python
# django/dispatch/dispatcher.py
...
class Signal:
    ...
    def send(self, sender, **named):
        ...
        return [
            (receiver, receiver(signal=self, sender=sender, **named))
            for receiver in self._live_receivers(sender)
        ]
    ...
```

The `send` method returns a list of tuples where the first element is the receiver (function) and the second element is the response from calling the receiver. `Signal` has an internal method, `_live_receivers`, that returns a list of active receivers for the specified `sender`.

The implementation does this with a simple list comprehension.

One issue with calling the `send` method to trigger the signal is that if one of the receivers raises an exception, there is no guarantee that all of the other receivers would be executed. This is where the `send_robust` method can help.

The `send_robust` method does the same thing as the `send` method except that exceptions are caught and added to the return value for later evaluation. This method guarantees that all receivers will be allowed to execute.

```python
# django/dispatch/dispatcher.py
...
class Signal:
    ...
    def send_robust(self, sender, **named):
        ...
        responses = []
        for receiver in self._live_receivers(sender):
            try:
                response = receiver(signal=self, sender=sender, **named)
            except Exception as err:
                responses.append((receiver, err))
            else:
                responses.append((receiver, response))
        return responses
    ...
```

Notice here that where the `send` method had a simple list comprehension, the `send_robust` method has a for loop that uses a try-except to make sure that every receiver is called and added to the responses list along with either the response from calling the receiver or the resulting exception that was raised.

Internally Django only uses the `send` method for sending signals. The `send_robust` can be used for your own custom signals.

#### Where are `request_started` and `request_finished` called from?

We know where the `request_started` and `request_finished` signals are created (`django.core.signals`) but where are those signals triggered (sent)?

Django sends the `request_started` signal from the `django.core.handlers.wsgi.WSGIHandler` `__call__` method ([code](https://github.com/django/django/blob/master/django/core/handlers/wsgi.py#L135)). On the line 2 of the `__call__` method you can see the call to `send`. The `WSGIHandler` class is the main entry point for Django applications. After an instance is instantiated, the `__call__` method gets called for each request. The `request_started` signal is essentially the first thing to be called on each request.

Django sends the `request_finished` signal from the `django.http.response.HttpResponseBase` classes `close` method ([code](https://github.com/django/django/blob/master/django/http/response.py#L27)). The `close` method ([code](https://github.com/django/django/blob/master/django/http/response.py#L238)) is called by the WSGI server on the response. When that happens the `request_finished` signal is sent.

```python
# django/http/response.py
...
class HttpResponseBase:
    ...
    def close(self):
        ...
        signals.request_finished.send(sender=self._handler_class)
```
We are obviously leaving a lot of things out here, but the important part is that the very last thing `close` does is send the `request_finished` signal.

### 2.2 Language Features


#### for-else block
You may have noticed some strange code in the examples above, a for loop with an associated else block (in the discussion about `django.dispatch.dispatcher.Signal`). This is not a common construct, but it exists in Python and its meaning is a little tough for some to remember.

```python
letters = ['a', 'b', 'c']
looking_for = 'a'

for letter in letters:
	if letter == looking_for:
		print('Found the letter!')
		break
else:
	print("Didn't find the letter.")
```

I think the above example illustrates its use nicely. The else block gets executed with the whole for loop completes and the break statement isn't reached.

#### Context manager

A lot of times in code you have to acquire a resource, do something with it, and then clean things up afterwards. This is the case when working with files (open, process, close), locks (lock, process, release), and many other resources. Because it is such a common flow (run common setup code, do something, run tear down code) Python added a statement that makes this a lot nicer.

```python
# working with files

# the standard method
file = open('tmp.txt')
...  # processing the file
file.close()

# context manager method
with open('tmp.txt') as file:
	...  # processing the file
```
The above code shows two methods of working with files. Both methods work just fine in Python. However the context manager method (using the `with` statement) is safer. The `with` statement in this context guarantees that the file will be closed, even if an exception is raised in the processing code.

We can see another use of context managers in the `django.dispatch.dispatcher.Signal` classes `connect` method. In there a lock is acquired and released using the `with` statement. Once again, this has the benefit of being safer because the lock is guaranteed to be released even if an exception is raised in the `with` block code.

## 3. Hands-on Exercises

### 3.1 Exercise 1

**Implement `todo_done` and `todo_undone` signals. Call the send method from the methods on the `Todo` model such as `mark_done` and `mark_undone`. Hook into the signal a callback that logs the item that was completed or undone.**

#### Hints

* We did something very similar to this in the Basics section above.

#### Possible Solution

```python
# todo/signals.py
import django.dispatch

todo_done = django.dispatch.Signal(providing_args=['item'])
todo_undone = django.dispatch.Signal(providing_args=['item'])
```

```python
# todo/models.py
...
from .signals import todo_done, todo_undone
...
class Todo(models.Model):
    ...
    def mark_done(self):
        if not self.done:
            self.done = True
            self.save()
            todo_done.send(sender=self.__class__, item=self)

    def mark_undone(self):
        if self.done:
            self.done = False
            self.save()
            todo_undone.send(sender=self.__class__, item=self)
...
```

```python
# todo/tasks.py
...
def log_todo_action(sender, item, **kwargs):
    if item.done:
		logger.info(f'Item complete: {item.item}')
	else:
		logger.info(f'Item undone: {item.item}')
```

```python
# todo/app.py
...
from . import signals, tasks
...
class TodoConfig(AppConfig):
	...
	def ready(self):
		signals.todo_done.connect(tasks.log_todo_action)
		signals.todo_undone.connect(tasks.log_todo_action)
```

__Note__: This is just one solution for solving this problem. The problem can be solved correctly in many different ways.

## 4. Contribute

### Resources

* [Django Signals documentation](https://docs.djangoproject.com/en/1.11/topics/signals/)
* [Signals open tickets](https://code.djangoproject.com/query?status=assigned&status=new&summary=~signal&col=summary&col=status&col=owner&col=type&col=version&col=has_patch&col=needs_docs&col=needs_tests&col=needs_better_patch&col=easy) (refer to [Understanding Django's Ticketing System]() for more details)

