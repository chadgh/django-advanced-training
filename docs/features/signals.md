# Signals

## 1. Basics
TODO

Basic example of setting up a signal (request_started and request_finished)
Slightly more complicated setup with a post_save signal on the Todo model.
Signals are like event-based programming. You can hook up callback functions that get executed when specific events happen.
Introduction to a few types of Signals available: pre_save, post_save, pre_delete, post_delete, m2m_changed on models. Django sends request_started and request_finished signals at the beginning and end of each request. The contrib.auth application also provides user_logged_in, user_logged_out, and user_login_failed.
Any django application can create and raise custom signals.

## 2. Deep Dive

### 2.1 Code Walk-through

#### django.core.signals

TODO
request_started
instance of Signal
to setup a custom signal is as simple as providing an instance of Signal for other bits of code to connect to.

#### django.dispatch.dispatcher.Signal
TODO
connect
All this method really accomplishes is that is adds the receiver function to the list of receivers held by the signal (instance of Signal)
send
Line 176 - list comprehension that loops over all live receivers and calls them.
send_robust
Just like send except catching the exceptions and adding them in place of the valid responses - all receivers get the signal.

#### Where are request_started and request_finished called from?
TODO
django.core.handlers.wsgi.WSGIHandler.__call__ - request_started
django.http.response.HttpResponseBase.close - request_finished


### 2.2 Language Features

Else on for! django.dispatch.dispatcher.Signal.connect near the bottom
Context manager (with statement) with a lock. django.dispatch.dispatcher.Signal.connect near the bottom - Lock is used because Python built in list data structure is not thread safe - line 113

#### thing

TODO

### 2.3 Software Architecture Features

#### thing

TODO ???

## 3. Hands-on Exercises

### 3.1 Exercise 1

**Using the django-advanced-training-project, implement a todo_completed signal and call the send method from the mark_as_done method on the Todo model. Hook into the signal a callback that logs the item that was completed.**

#### Hints

* TODO

#### Possible Solution

TODO

## 4. Contribute

### Resources

TODO

* [Django class-based views documentation](https://docs.djangoproject.com/en/1.11/ref/class-based-views/)
* [Django class-based views reference](https://docs.djangoproject.com/en/1.11/topics/class-based-views/)
* [Classy Class-based Views](https://ccbv.co.uk)
* [Generic views open tickets](https://code.djangoproject.com/query?status=assigned&status=new&component=Generic+views&col=summary&col=status&col=owner&col=type&col=version&col=has_patch&col=needs_docs&col=needs_tests&col=needs_better_patch&col=easy) (refer to [Understanding Django's Ticketing System]() for more details)

