# Messages Framework

## Basics


Django's messages framework makes sending one-time messages to users very simple. After you get the messages framework setup in your Django project it is as simple as adding something like the following code in your views and updating your templates to display the messages:

```python
from django.contrib import messages
...
# in view code
messages.success(request, 'You successfully did something!')
```

## Deep Dive

## Hands-on Exercises

## Contribute

### Resources

* [contrib.messages documentation](https://docs.djangoproject.com/en/1.11/ref/contrib/messages/)
* [contrib.messages open tickets](https://code.djangoproject.com/query?status=assigned&status=new&component=contrib.messages&col=summary&col=status&col=owner&col=type&col=version&col=has_patch&col=needs_docs&col=needs_tests&col=needs_better_patch&col=easy) (refer to [Understanding Django's Ticketing System]() for more details)
