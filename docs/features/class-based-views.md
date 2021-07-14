# Class-based Views

## 1. Basics

<iframe width="560" height="315" src="https://www.youtube.com/embed/Co2pM7QZT5A" frameborder="0" allowfullscreen></iframe>

Django's class-based views provide an object-oriented (OO) way of organizing your view code. Most Django tutorials and training materials start developers off with the simple style of function-based views (which were available in Django long before class-based views). Class-based views were introduced to help make view code more reusable and provide for better view code organization.

The structure of a simple function-based view that is used to process both `GET` and `POST` requests might look like this:

```python
# views.py
def simple_function_based_view(request):
    if request.method == 'GET':
        ...  # code to process a GET request
    elif request.method == 'POST':
        ...  # code to process a POST request
```

The same thing with a class-based view could look like this:

```python
# views.py
from django.views import View

class SimpleClassBasedView(View):
    def get(self, request):
        ...  # code to process a GET request

    def post(self, request):
        ...  # code to process a POST request
```

Hooking up class-based views in the `urls.py` file is a little different too:

```python
# urls.py
...
from . import views

urlconfig = [
    # the function-based view is added as the second param
    url(r'^function/$', views.simple_function_based_view),
    # the as_view method is called on the class-based view and 
    # the result is the value of the second param
    url(r'^class/$', views.SimpleClassBasedView.as_view()),
]
```

To illustrate this further we will walk through converting a function-based view to a class-based view that does the same thing.

```python
# views.py
from datetime import datetime
from django.http import HttpResponse

def show_the_time(request):
    now = datetime.now()
    html = "<html><body>It is now {}</body></html>".format(now)
    return HttpResponse(html)
```

```python
# urls.py
from django.conf.urls import url

from . import views

urlpatterns = [
    url(r'^now/$', views.show_the_time),
]
```

In order to do the same thing using a class-based view the files would have to change as follows:

```python
# views.py
from datetime import datetime
from django.http import HttpResponse
from django.views import View  # import the View parent class

class ShowTimeView(View):  # create a view class

    # change the function-based view to be called get and add the self param
    def get(self, request):
        now = datetime.now()
        html = "<html><body>It is now {}</body></html>".format(now)
        return HttpResponse(html)
```

```python
# urls.py
from django.conf.urls import url

from . import views

urlpatterns = [
    url(r'^now/$', views.ShowTimeView.as_view()),  # change how we reference the new view.
]
```
Note that there are not many changes in order to change from one type of view (function-based) to the other (class-based). The benefit of going with the class-based view (even in this simple example) is that the view is going to be more robust. The class-based views (view classes that extend the `View` class) get built-in responses to unsupported methods (`POST`, `PUT`, `PATCH`, etc.) and get support for the `OPTIONS` HTTP method too. All that is required to support other HTTP methods is to implement the same named method in the view class.

There are also generic class-based views. These are class-based views that provide extra common functionality. For example, a common type of view might be called a template view: a view that generates some context and sends the context to a specified template for rendering. Django provides a generic class-based view for that very purpose, `TemplateView`.

To use the example above, we will assume that the html portion is in a template called `show_time.html`. If so, we can change the `ShowTimeView` class to extend from the `TemplateView` instead of `View` and get the benefits of the common code.

```python
# views.py
from datetime import datetime
from django.http import HttpResponse
from django.views.generic import TemplateView  # import the TemplateView parent class

class ShowTimeView(TemplateView):  # extend from TemplateView
    template_name = 'show_time.html'  # add a template_name attribute

    # change the get method to get_context_data
    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['now'] = datetime.now()
        return context
```
The `urls.py` file shouldn't have to be modified.

To learn more about using class-based views and generic class-based views see the Django documentation [ref](https://docs.djangoproject.com/en/1.11/ref/class-based-views/) and [topics](https://docs.djangoproject.com/en/1.11/topics/class-based-views/) sections.

## 2. Deep Dive: Code Walk-through

<iframe width="560" height="315" src="https://www.youtube.com/embed/Qki2m5AyfWw" frameborder="0" allowfullscreen></iframe>

### `View`
The class-based views in Django all extend from the parent class `View`. This class can be found in `django.views.generic.base` ([code here](https://github.com/django/django/blob/master/django/views/generic/base.py#L31)).

The `View` class has three methods that we will take a closer look at. For convenience the important parts of these methods are included below.

```python
# django/views/generic/base.py
...
class View:
    ...
    def __init__(self, **kwargs):
        ...
        for key, value in kwargs.items():
            setattr(self, key, value)

    @classonlymethod
    def as_view(cls, **initkwargs):
        ...
        def view(request, *args, **kwargs):
            self = cls(**initkwargs)
            if hasattr(self, 'get') and not hasattr(self, 'head'):
                self.head = self.get
            self.request = request
            self.args = args
            self.kwargs = kwargs
            return self.dispatch(request, *args, **kwargs)
        view.view_class = cls
        view.view_initkwargs = initkwargs
        ...
        return view

    def dispatch(self, request, *args, **kwargs):
        ...
        if request.method.lower() in self.http_method_names:
            handler = getattr(self, request.method.lower(), self.http_method_not_allowed)
        else:
            handler = self.http_method_not_allowed
        return handler(request, *args, **kwargs)

    ...
```

The `__init__` method is fairly simple. It takes keyword arguments and sets each of them as attributes on the instance with the values provided. A Django developer doesn't have a need to instantiate class-based views directly, so the constructor isn't something that you would interact with directly, but it is used by the `as_view` method to provide a chance when calling `as_view` to override attributes then.

The `dispatch` method contains the actual view logic. It takes a request, finds the method that should be called for the given request (by using the HTTP method used), and returns the results of calling that method. If there isn't a method for the HTTP method used, the default view is the `http_method_not_allowed` view. It returns an HTTP 405 response.

The `as_view` method creates and returns a new function that is used as a Django view. Here we can see that even class-based views are simply just function-based views when they are actually used.

The `view` function that is created and returned from the `as_view` method, looks a lot like a function-based view. It takes a request and returns the results of calling the class-based views `dispatch` method. It is also interesting to see that the `view` function on each request will instantiate a new instance of the view class, set the request, args, and kwargs attributes, and then call and return the results from the view instance's `dispatch` method. It is also interesting to note that the `view` function also gets a few attributes of its own including the `view_class` and `view_initkwargs` attributes.

### `ContextMixin` and `TemplateResponseMixin`

The suffix `Mixin` in Python is a convention that is often used to signify that the class provides functionality that can be used by other things. These types of classes aren't usually instantiated or used on their own other than to be extended from so that other classes can use the functionality they provide. These types of classes usually have no parent class and provide a very specific bit of functionality.

The `ContextMixin` class found in the [generic views code](https://github.com/django/django/blob/master/django/views/generic/base.py#L16) provides child classes with a `get_context_data` method that can be used.

```python
class ContextMixin:
    ...
    def get_context_data(self, **kwargs):
        if 'view' not in kwargs:
            kwargs['view'] = self
        if self.extra_context is not None:
            kwargs.update(self.extra_context)
        return kwargs
```

By itself, the `ContextMixin` isn't that helpful. The `get_context_data` method makes sure that there is a value `view` in kwargs and also makes sure that if there is anything in the `extra_context` attribute it is also added to kwargs before returning kwargs. In the next section we will see why this mixin is helpful, but first we will take a looks at the `TemplateResponseMixin`.

The `TemplateResponseMixin` ([also in the same file](https://github.com/django/django/blob/master/django/views/generic/base.py#L109)) looks like this:

```python
...
class TemplateResponseMixin:
    ...
    template_name = None
    ...
    def render_to_response(self, context, **response_kwargs):
        ...
        response_kwargs.setdefault('context_type', self.context_type)
        return self.response_class(
            request=self.request,
            template=self.get_template_names(),
            context=context,
            using=self.template_engine,
            **response_kwargs
        )
    ...
```

This class provides a few more attributes and another method that are not seen here, but what is shown is the important part for understanding what this mixin provides. The `render_to_response` method takes a context dict then instantiates a response object and returns the response object. It also takes care of defining which template should be used, as denoted by the `template_name` attribute.

These two mixins don't seem to provide very much, but when combined into the same view class, we can begin to see the power they offer. In the next section we look at the `TemplateView` class-based view and what it provides.

### `TemplateView`

The `TemplateView` class-based view provides a simple reusable (generic) class-based view for rendering views that rely on context data and rendering a template. We look at it below ([code on github](https://github.com/django/django/blob/master/django/views/generic/base.py#L145)).

```python
class TemplateView(TemplateResponseMixin, ContextMixin, View):
    ...
    def get(self, request, *args, **kwargs):
        context = self.get_context_data(**kwargs)
        return self.render_to_response(context)
```
As we can see the `TemplateView` provides a default `get` implementation that builds the context, by calling the `get_context_data` method provided by the `ContextMixin` class and then returns the results of calling the `render_to_response` method provided by the `TemplateResponseMixin` class. If you want to use the `TemplateView` all you should have to do is extend it and define the `template_name` attribute as well as an extended definition for the `get_context_data` method. For example:

```python
class ShowTimeView(TemplateView):
    template_name = 'show_time.html'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['now'] = datetime.now()
        return context
```
The `ShowTimeView` will take care of rendering the proper response object for us and we have no need of defining a `get` method because we extended the `TemplateView` class-based view.

## 3. Deep Dive: Language Features

### First-class functions

In Python, functions are known as [first-class](https://en.wikipedia.org/wiki/First-class_function) or higher order functions. Essentially this means that functions can be passed to other function calls, returned from other functions, and be assigned to variable names.

We see an [example of the use of higher order functions](https://github.com/django/django/blob/master/django/views/generic/base.py#L62) in the `View` class's `as_view` method. When that method is called, it generates (defines) a new function called `view` and returns it. This means that the urls configuration gets a function where it is expecting a function:

```python
    url(r'^$', views.ShowTimeView.as_view()),
```

With function-based views, the second argument is the function itself that should be executed when the url regex is matched. For class-based views we call another function, `as_view`, which returns a function (instead of naming it explicitly).

Let's look at another example of using functions as first-class citizens:

```python
>>> def make_power_func(power):
...     def to_the_power(number):
...         return number ** power
...     return to_the_power
>>> pow_10 = make_power_func(10)
>>> pow_10(2)
1024
```

In this example the `make_power_func` function takes a number, `power`, and creates a new function that encapsulates the `power` value. The `make_power_func` function also returns the newly defined `to_the_power` function. We then assign that function to the value of `pow_10` and we can then call the function through that new name. We could call the `make_power_func` function for as many values as we wish and create a power function on the fly.

### Multiple inheritance

Python's OOP model allows for multiple inheritance. This can be easily seen in the `TemplateView` class above:

```python
class TemplateView(TemplateResponseMixin, ContextMixin, View):
```

When defining a class the parents of that class are defined in parentheses after the class name. The `TemplateView` class has three explicit parent classes. There would also most likely be some implicit parent classes (such as `object` which is the parent of all classes in Python, but rarely explicitly defined). The order of the parents matters here because when the `super` function is called in any method in order to call a parent method, parents are searched in the order defined. So when we call the `get_context_data` method on the `TemplateView` class instance, it first looks in `TemplateResponseMixin` and then moves to `ContextMixin` (before stopping because the method was found there).

In order to see the parent classes, and the order that they will be called/searched in, you can call the `mro` method of a class (MRO stands for Method Resolution Order).

```python
>>> from django.views.generic import TemplateView
>>> TemplateView.mro()
[<class 'django.views.generic.base.TemplateView'>, <class 'django.views.generic.base.TemplateResponseMixin'>, <class 'django.views.generic.base.ContextMixin'>, <class 'django.views.generic.base.View'>, <class 'object'>]
```
This parent list is a predictable and specifically ordered list of parents that defines the order that parents are searched in when calling a method and using `super`.

## 4. Deep Dive: Software Architecture Features

### Multiple inheritance

Multiple inheritance can be a tricky problem to solve, what do you do if multiple parents define the same method, which one do you call if the method is called from a child class instance?

One algorithm that solves this problem (and the algorithm that is used for MRO in Python) is the [C3 linearization]() algorithm. The C3 linearization algorithm provides a consistent order to the MRO list of parent classes because of this it is deterministic in the order of the parent classes it produces.

## 5. Hands-on Exercises

**Implement a generic class-based view called `StaticView` that extends from `View` and provides a get method that will respond with a static HTML page.**

### Hints

* Make sure it is reusable. The static HTML should to be specified by each child class-based view of `StaticView`.
* Take a look at what the `TemplateView` does in its `get` method.

### Possible Solution

There are most definitely other ways to solve this problem, but here is one simple solution.

```python
from django.http import HttpResponse
from django.views import View


class StaticView(View):
    static_html = None

    def get(self, request):
        html = open(self.static_html).read()
        return HttpResponse(html)


class MyStaticView(StaticView):
    static_html = 'static_file.html'
```

The `StaticView` class can be extended and the `static_html` attribute updated. The `urls.py` file would have an entry for the `MyStaticView` child class. Because of the way `View` is implemented and the `as_view` method, we can also add an entry to our urls that looks like this.

```python
    url(r'^test/$', views.StaticView.as_view(static_html='test.html'))
```

## 6. Contribute

### Resources

* [Django class-based views documentation](https://docs.djangoproject.com/en/1.11/ref/class-based-views/)
* [Django class-based views reference](https://docs.djangoproject.com/en/1.11/topics/class-based-views/)
* [Classy Class-based Views](https://ccbv.co.uk)
* [Generic views open tickets](https://code.djangoproject.com/query?status=assigned&status=new&component=Generic+views&col=summary&col=status&col=owner&col=type&col=version&col=has_patch&col=needs_docs&col=needs_tests&col=needs_better_patch&col=easy) (refer to [Understanding Django's Ticketing System]() for more details)

