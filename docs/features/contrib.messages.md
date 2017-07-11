# Messages Framework

## 1. Basics

Django's messages framework makes sending one-time messages to users simple. After setting up the messages framework in your Django project (which is setup by default in the standard Django project) it is as simple as adding a message with a single call in your views. 

```python
# views.py
from django.contrib import messages
...
def some_view(request):
    ...
    # there are two ways to add a message in view code
    # (a) using the add_message method, 
    messages.add_message(request, messages.INFO, 'Hello!')

    # (b) using one of the convenience methods for the message level (info in this case).
    messages.info(request, 'Hello!')
    ...
...
```

The above code adds the message "Hello!" to the messages storage backend to be displayed to the user later during the current request or during the next request. Messages are stored with the level information attached to them for later use. The messages above are stored with the level of 'INFO'. When a message is displayed it is removed from the storage backend.

In order to display messages to the user the messages need to be looped over in a template. The following template code is one way to do this:

```djangohtml
<!-- base.html -->
...
{% if messages %}
<ul class="messages">
    {% for message in messages %}
    <li{% if message.tags %} class="{{ message.tags }}"{% endif %}>{{ message }}</li>
    {% endfor %}
</ul>
{% endif %}
...
```

Messages are added to the messages storage at specific levels. These levels are similar to the logging levels in Python, but are not the same. Each message level corresponds to an integer value. Below are the default levels and their values along with the tag that will be attached (which is also the convenience method names).

| Level Constant | Tag     | Value |
| -------------- | ------- | ----- |
| `DEBUG`        | debug   | 10    |
| `INFO`         | info    | 20    |
| `SUCCESS`      | success | 25    |
| `WARNING`      | warning | 30    |
| `ERROR`        | error   | 40    |

The messages framework is setup by default in the standard Django project that you get from running `startproject` with `django-admin`. If you have setup your project using another method you might need to make the following changes in your code in order to make the messages framework available.

1. Make sure `django.contrib.messages` is in the `INSTALLED_APPS` list in your settings.
2. Make sure `django.contrib.sessions.middleware.SessionMiddleware` and `django.contrib.messages.middleware.MessageMiddleware` are both in the `MIDDLEWARE` list in your settings. Note that the session middleware needs to come before the messages middleware.
3. Make sure the `context_processors` list in the `TEMPLATES` settings contains `django.contrib.messages.context_processors.messages`.

For more information about Django's messages framework see the [Django documentation](https://docs.djangoproject.com/en/1.11/ref/contrib/messages/). Also a [Google search](https://www.google.com/search?q=django+messages+framework) for `django messages framework` results in many helpful resources for learning more.

## 2. Deep Dive

### 2.1. Code Walk-through

#### `MessageMiddleware`
The messages middleware is the place to start as we take a closer look at the implementation for this feature. The middleware is one of the things that is required to make sure you are able to use the messages framework. The code can be found [on github](https://github.com/django/django/blob/master/django/contrib/messages/middleware.py).

There are two methods in the `MessageMiddleware` class that are executed for each request/response. The first is the `process_request` method which is executed on each request as it comes into Django.

```python
def process_request(self, request):
    request._messages = default_storage(request)
```

This method adds a new `_messages` attribute to the request object for the current request. The value of that attribute is the return value from calling the `default_storage` function. The `default_storage` function is also very simple, it instantiates the messages storage backend, set in the projects settings, with the request object ([`default_storage` code](https://github.com/django/django/blob/master/django/contrib/messages/storage/__init__.py)).

The other method on the `MessageMiddleware` class we need to take a look at is the `process_response` method.

```python
def process_response(self, request, response):
    ...
    # A higher middleware layer may return a request which does not contain
    # messages storage, so make no assumption that it will be there.
    if hasattr(request, '_messages'):
        unstored_messages = request._messages.update(response)
        ...
    return response
```

Before each response is sent back from Django to the user's browser the response is sent to the `update` method of the messages storage backend instance. The implementation for this method is found in the `BaseStorage` class that is extended from for each of the message storage backends ([see the code here](https://github.com/django/django/blob/master/django/contrib/messages/storage/base.py#L115)). This method makes sure that any messages that where not read for this response will be saved/stored for later requests. The actual implementation of what that means depends on the messages storage backend implementation. The `_store` method is called from the `update` method and is storage backend dependent.

The end result of this means that before a response is sent back to the user's browser, all unread messages are stored for later use.

#### `messages.context_processors`

The messages context processor ([code](https://github.com/django/django/blob/master/django/contrib/messages/context_processors.py)) makes sure that every context that is sent to a template to be rendered gets the following context variables added:

<dl>
<dt><code>messages</code></dt>
<dd>A list of the messages that haven't been displayed yet.</dd>

<dt><code>DEFAULT_MESSAGE_LEVELS</code></dt>
<dd>A dict of the default message levels where the keys are the level names and the values are the level integer values (similar to the message level table above).</dd>
</dl>

#### Message Storage

The messages framework stores the messages in what is known as a storage backend. These backends must extend from the `BaseStorage` class found in [storage/base.py](https://github.com/django/django/blob/master/django/contrib/messages/storage/base.py#L43) in the messages framework code. The `BaseStorage` class' doc string states that all children classes that implement a new storage backend must implement two methods: `_get` and `_store`.

```python
# message storage backend _get method signature.
def _get(self, *args, **kwargs):
    """
    Retrieve a list of stored messages. Return a tuple of the messages
    and a flag indicating whether or not all the messages originally
    intended to be stored in this storage were, in fact, stored and 
    retrieved; e.g., ``(messages, all_retrieved)``.
    """
    ...
    
# messages storage backend _store method signature.
def _store(self, messages, response, *args, **kwargs):
    """
    Store a list of messages and return a list of any messages which could
    not be stored.
    """
    ...
```

The `_get` method must return a tuple where the first element is a list of the stored messages and the second element is a flag indicating whether or not all of the messages where stored and retrieved.

The `_store` method must store a list of messages and return a list of the messages that couldn't be stored.

The `BaseStorage` class also provides two main methods that define how all storage backends work. First the [`add` method](https://github.com/django/django/blob/master/django/contrib/messages/storage/base.py#L129). This method is responsible for adding messages to the messages system. It takes a message level, the message and optional extra tags, instantiates a new `Message` and appends that message to an internal list of messages (this is only adding the messages in memory, actual storage in the backend happens later when the `update` method is called from the `process_response` method of the `MessagesMiddleware`, described above).

```python
def add(self, level, message, extra_tags=''):
    ...
    if not message:
        return

    if level < self.level:
        return
    
    self.add_new = True
    message = Message(level, message, extra_tags=extra_tags)
    self._queued_messages.append(message)
```

The first thing we can see is that the `add` method returns right away and doesn't queue anything or even raise errors if the `message` argument is falsy or the `level` argument is less than the current message level set. Otherwise, a new `Message` instance is created and appended to the `_queued_messages` attribute of the storage backend.

From this we can see that messages are not stored based on the message level set. Often it is assumed that messages are stored and when displaying them the message level can be set to determine which ones get shown, but that is not the case as we can see from the implementation.

The `update` method is called by the `MessagesMiddleware`'s `process_response` method. It is responsible for calling the messages storage backend's `_store` method.

```python
def update(self, response):
    ...
    if self.used:
        return self._store(self._queued_messages, response)
    elif self.added_new:
        messages = self._loaded_messages + self._queued_messages
        return self._store(messages, response)
```

The `self.used` is set to true on the storage backend instance if the messages have been iterated over. The `self.added_new` is set to true inside the `add` method that we saw above. So the `update` method only stores the queued messages if the backend was iterated over, otherwise it stores everything (`_loaded_messages` contains messages that were stored during a previous request in the storage backend).

All messages are stored by instantiating the [`Message` class](https://github.com/django/django/blob/master/django/contrib/messages/storage/base.py#L7). This class takes in its constructor the level, message, and extra tags. The message argument is cast to a string if the `Message` instance is cast to a string (see the `__str__` method on `Message`) or when the messages are iterated over (see the `_prepare` method on `Message`). Seeing this in the implementation means that we can store in the messages framework objects other than strings as long as they can be cast to a string and that casting results in the messages that we want stored.

In order to get a better understanding of how the messages storage backends work, we can take a look at one of the more simple storage backends provided by Django, the [`SessionStorage` class](https://github.com/django/django/blob/master/django/contrib/messages/storage/session.py#L10). Like we already learned, a storage backend only needs to implement the `_get` and `_store` methods, so we should be able to see how the `SessionStorage` backend is implemented by just looking at those two methods.

```python
...
class SessionStorage(BaseStorage):
    ...
    session_key = '_messages'
    ...
    def _get(self, *args, **kwargs):
        ...
        return self.deserialize_messages(self.request.session.get(self.session_key)), True

    def _store(self, messages, response, *args, **kwargs):
        ...
        if messages:
            self.request.session[self.session_key] = self.serialize_messages(messages)
        else:
            self.request.session.pop(self.session_key, None)
        return []
    ...
```

The `SessionStorage` backend does make use of a few helper methods to serialize and deserialize the messages before and after storing them in the session, but other than that the session storage backend implementation is fairly straight forward. The `_get` method pulls any messages out of the session (`self.request.session.get(self.session_key)`) and the`_store` method puts messages into the session (`self.request.session[self.session_key] = messages`).

#### Messages API and Constants
The final piece to the messages framework puzzle is the interface used by the Django developer. When you import `from django.contrib import messages` you are importing everything in the [`constants.py`](https://github.com/django/django/blob/master/django/contrib/messages/constants.py) file and the [`api.py`](https://github.com/django/django/blob/master/django/contrib/messages/api.py) file.

The `constants.py` file contains the message level constants that are used throughout the code. 

The `api.py` file contains the `add_message` function, `set_level` function, and the convenience functions for adding messages at the specific levels (`debug`, `info`, `success`, `warning`, `error`).

The `add_message` function is the gateway function. In most Django developer code, the `add_message` function is the only thing that is specifically interacted with in the framework.

```python
...
def add_message(request, level, message, extra_tags='', fail_silently=False):
    ...
    try:
        messages = request._messages
    except AttributeError:
        ...  # code here uses the fail_silently argument to determine if an exception is raised
    else:
        return messages.add(level, message, extra_tags)
...
```

Remember that the `_messages` attribute on the request object is an instance of the storage backend. So the `add_message` function simply delegates to the storage backend's `add` method as seen previously.

The `set_level` function allows you to change the minimum message level that should be store.

```python
def set_level(request, level):
    ...
    if not hasattr(request, '_messages'):
        return False
    request._messages.level = level
    return True
```
This function will return `True` or `False` indicating if it was able to change the message level. It simply changes the level attribute on the storage backend instance found in the `_messages` attribute of the request object.

Finally, there are several convenience functions provided to make it simpler to add messages at specific levels. These simple functions all do the exact same thing.

```python
def debug(request, message, extra_tags='', fail_silently=False):
    """Add a message with the ``DEBUG`` level."""
    add_message(request, constants.DEBUG, message, extra_tags=extra_tags, 
                fail_silently=fail_silently)
```
All of the convenience functions are the same here. The only differences being the name, the doc string, and which constant level value that is provided as the second argument of the `add_message` call.

### 2.2 Language Features

#### try-except-else
Python has the concept of an `else` block in connection with a `try-except` block. This is for code that should be executed if no exceptions were raised.

```python
try:
    ...  # some code that could raise an exception
except Exception as e:
    ...  # do something when an exception is raised
else:
	...  # code that is executed if the try block completes and no exceptions were raised
```

We saw this in the `api.py` file in the `add_message` function. The `try` block was used to get access to the current instance of the message storage backend (`request._message`) and so the `else` block is executed if that access doesn't fail.

#### Custom containers

Python has double underscore (dunder) methods that can be defined on classes to make them work within built-in constructs and operators. You can create your own class and if you also define the `__len__`, `__iter__`, and `__contains__` methods you have also defined a new type of container object.

The `__len__` method is used to return the length of the container. This is used by the built-in `len` function.

The `__iter__` method is used to return an iterator over the contents of the container. This is used by looping constructs.

The `__contains__` method is used to determine if an object is in the container. This is used by the `in` operator.

The `BaseStorage` class implements these methods and it is why we can iterate over the storage backend instance in the templates and determine who many messages there are.


### 2.3 Software Architectural Features

#### Inheritance

In object oriented programming (OOP) inheritance is used to share code between objects and build upon the functionality provided in other classes. In Python this syntax is:

```python
class MyBackend(BaseStorage):
    ...
```

This means that the `MyBackend` class extends the `BaseStorage` class. All of the methods and attributes defined on the `BaseStorage` class are now available to the `MyBackend` class. In this example the `BaseStorage` class is referred to as the parent class of `MyBackend`.

In the messages framework we see the use of inheritance by defining the `BaseStorage` parent class that all the child classes must inherit from. There is no need to write any other code than the two methods that must get implemented (`_get` and `_store`). All other interface code is shared among all storage backends, because they must all extend from the same parent class.

#### Singleton pattern

In OOP it is sometimes common to make sure there is only ever one instance of a class. This is called the [Singleton pattern](https://en.wikipedia.org/wiki/Singleton_pattern). Code is usually written in the class constructors to make sure that it is only able to be instantiated once.

In Python this can be accomplished by simply writing a module (file). Even if the module is imported more than once of under different names, there will only be one instance of it. This acts as a singleton.

The messages API is implemented as a singleton. You import `from django.contrib import messages`, which pulls in the module. Functions defined in the `api.py` file are accessed as methods on an instance.

To illustrate this more simply we can do the following. We will create a file called `single.py` with the following contents.
```python
x = 1


def calc(num):
    return num * x
```
Then we will start our REPL and do the following.
```python
>>> import single
>>> single.x
1
>>> single.calc(2)
2
>>> single.x = 10
>>> single.x
10
>>> single.calc(2)
20
```
Note that `single` here looks just like an instance of a class with attributes that can be changed and methods that can be called.

Now, in that same REPL session, if we import the `single` module again but using a different name, we can see that we will always just have one instance of the module.

```python
...
>>> import single as s
>>> s.x
10
```

## 3. Hands-on Exercises

### 3.1 Exercise 1

**Implement a new messages storage backend that uses Django's caching framework.**

The hints section below is to help you come up with a solution to this problem. The possible solution section below shows one possible solution to this problem. Try out your own solution first with the help of the hints before taking a look at the possible solution section.

#### Hints

* You can get access to the low-level Django cache framework by importing it `from django.core.cache import cache`. See Django's [low-level cache documentation](https://docs.djangoproject.com/en/1.11/topics/cache/#the-low-level-cache-api).
* You can set and get values from the cache framework using the `set` and `get` methods.
* Remember that you probably want to store messages on a per user basis.
* Don't forget to change the `MESSAGE_STORAGE` setting to point to your custom backend.

#### Possible Solution

Here is one possible solution. One major problem with this solution is that it will only store messages in the cache based on the username of the user. This is a problem in that if you are storing messages in views that don't require authentication, than any user that is not authenticated will have a blank username and all messages will be stored in the same place for those users. So, this implementation is not perfect, but it will work in authenticated views.

```python
from django.contrib.messages.storage.base import BaseStorage
from django.core.cache import cache


class CacheStorage(BaseStorage):

    def cache_key(self):
        return f'messages-for-{self.request.user.username}'

    def _get(self, *args, **kwargs):
        return (cache.get(self.cache_key()) or [], True)

    def _store(self, messages, response, *args, **kwargs):
        if messages:
            cache.set(self.cache_key(), messages)
        else:
            cache.set(self.cache_key(), [])
        return []
```

## 4. Contribute

### Resources

* [contrib.messages documentation](https://docs.djangoproject.com/en/1.11/ref/contrib/messages/)
* [contrib.messages open tickets](https://code.djangoproject.com/query?status=assigned&status=new&component=contrib.messages&col=summary&col=status&col=owner&col=type&col=version&col=has_patch&col=needs_docs&col=needs_tests&col=needs_better_patch&col=easy) (refer to [Understanding Django's Ticketing System]() for more details)
