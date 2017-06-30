# Messages Framework

## Basics

Django's messages framework makes sending one-time messages to users simple. After settings up the messages framework in your Django project it is as simple as adding a message with a single call in your views. 

```python
# views.py
from django.contrib import messages
...
# in view code
messages.add_message(request, messages.INFO, 'Hello!')
```

This code will add the message "Hello!" to the messages storage backend to be displayed to the user on another request. Messages are stored with the level information attached to them for later use. The message above was stored with the level of 'INFO'.

One way of displaying the messages to the user is by looping over the messages in a template:

```djangohtml
# base.html
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

Messages are added to the messages storage at specific levels. These levels are similar to the logging levels in Python, but are not the same. Each message level corresponds to an integer value. Below are the default levels and their values.

| Level Constant | Tag     | Value |
| -------------- | ------- | ----- |
| `DEBUG`        | debug   | 10    |
| `INFO`         | info    | 20    |
| `SUCCESS`      | success | 25    |
| `WARNING`      | warning | 30    |
| `ERROR`        | error   | 40    |

In order to setup your Django project to be able to use the messages framework you need to take the following steps:

1. Make sure `django.contrib.messages` is in the `INSTALLED_APPS` list in your settings.
2. Make sure `django.contrib.sessions.middleware.SessionMiddleware` and `django.contrib.messages.middleware.MessageMiddleware` are in the `MIDDLEWARE` list in your settings.
3. Make sure the `context_processors` list in the `TEMPLATES` settings contains `django.contrib.messages.context_processors.messages`.

For more information about Django's messages framework see the [Django documentation](https://docs.djangoproject.com/en/1.11/ref/contrib/messages/). Also a [Google search](https://www.google.com/search?q=django+messages+framework) results in many helpful resources for learning more.

## Deep Dive

#### `MessageMiddleware`
The messages middleware is the place to start as we take a closer look at the implementation for this feature. The middleware is one of the things that is required to make sure you are able to use the messages framework. We can see the [code on github](https://github.com/django/django/blob/master/django/contrib/messages/middleware.py).

There are two methods in the `MessageMiddleware` class that are executed for each request/response. The first is the `process_request` method which is executed on each request as it comes into Django.

```python
def process_request(self, request):
	request._messages = default_storage(request)
```

This method adds a new `_messages` attribute to the request object. The value of that attribute is the return value from calling the `default_storage` function. The `default_storage` function is also very simple, it instantiates the messages storage backend, set in the projects settings, with the request object [`default_storage`](https://github.com/django/django/blob/master/django/contrib/messages/storage/__init__.py).

The other method on the `MessageMiddleware` class to understand is the `process_response` method.

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

Before each response is sent back from Django to the user's browser the response is sent to the `update` method of the messages storage backend instance. This method is actually found in the `BaseStorage` class that is extended from for each of the message storage backends ([see the code here](https://github.com/django/django/blob/master/django/contrib/messages/storage/base.py#L115)). This method makes sure that any messages that where not read for this response will be saved/stored for later requests. The actual implementation of what that means depends on the messages storage backend implementation. The `_store` method is called from the `update` method and is storage backend dependent.

#### `messages.context_processors`

Al


## Hands-on Exercises

### Exercise 1

**Implement a new messages storage backend that uses Django's caching framework.**

#### Possible Solution

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

Then set the `MESSAGE_STORAGE` setting to `path.to.the.class.CacheStorage`.

This backend 


## Contribute

### Resources

* [contrib.messages documentation](https://docs.djangoproject.com/en/1.11/ref/contrib/messages/)
* [contrib.messages open tickets](https://code.djangoproject.com/query?status=assigned&status=new&component=contrib.messages&col=summary&col=status&col=owner&col=type&col=version&col=has_patch&col=needs_docs&col=needs_tests&col=needs_better_patch&col=easy) (refer to [Understanding Django's Ticketing System]() for more details)
