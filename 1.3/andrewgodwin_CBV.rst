Class Based Views
=================

Class-based views are probably the biggest new feature to land in Django 1.3,
and the one that will probably make the biggest impact on the way you write and
structure your views. They're intended to increase code reuse, and to make
third-party apps easier to extend and integrate into your project.

Still, for those who are unfamiliar with class-based views (which, I suspect,
is a good portion of the Django user base right now), you may be wondering what
they are, exactly, and why they're apparently the solution to all of your
problems [#]_.

.. [#] They're not a panacea, so don't listen to anyone who says they'll solve all problems, ever. Still, they're really quite useful in a range of situations.

Views: a history
----------------

Ever since Django was first released, the concept of a Django view has been
pretty stable; as the oft-quoted phrase goes, a view is *a function which takes
a request and returns a response*.

Except, of course, that's not quite true - they're *callables* which take
a request and return a response. Since this is Python, functions aren't the
only thing which are callable - in particular, if you define a ``__call__``
method on a class, instances of that class become callable (the class itself
already is, of course - when you call it, you make a new instance of that
class).

This simple fact means that you can make views which are actually instances of
classes, rather than functions, such as this simple example

.. sourcecode:: python

    from django.http import HttpResponse

    class MyView(object):

        def __init__(self, noun="world"):
            self.noun = noun

        def get_message(self):
            return "Hello, %s!" % self.noun

        def __call__(self, request):
            return HttpResponse(self.get_message())

You can see how instances of this class are views - and moreover, how different
instances can easily be configured differently, which brings us to the concepts
behind class-based views, and in particular, class-based generic views.

Generic Views
-------------

Generic views have also been with Django for a long time; some of you may be
familiar with them, others probably aren't. In short, they're simple views that
provide very common functionality - displaying a template, performing
a redirect, displaying a single object by ID, that sort of thing.

Obviously, to be generic a view has to be reasonably flexible, and generic
views accomplished this by having a dizzying array of options, which you could
supply using the third argument in URLconfs (which, for the unfamiliar, is
a dictionary of keyword arguments to pass to the view function).

This is fine for simpler views, but even as you start getting remotely flexible
(in order to integrate well with a variety of codebases), one must provide
a variety of options. This, for example, is the full function signature of
``object_detail``, a relatively simple view which takes a url like
``year/month/day/slug`` and shows you the item with a matching slug from
a queryset

.. sourcecode:: python

    def object_detail(request, year, month, day, queryset, date_field,
        month_format='%b', day_format='%d', object_id=None, slug=None,
        slug_field='slug', template_name=None, template_name_field=None,
        template_loader=loader, extra_context=None, context_processors=None,
        template_object_name='object', mimetype=None, allow_future=False):

As you can see, there really are quite a lot of options there. While only a few
of them are needed in each project, it's a different set for each project.
Because the easiest way to reuse code inside a function is to add an option,
the function signature ends up being very complex.

Python has a solution to reusing code flexibly, along with many other languages
- classes (and object inheritance), which is where class-based views come in.

Reusing code
------------

Because we can define overrideable attributes and methods on classes, there's
no need for such a large function signature. Where we had a ``template_name``
argument before, we can replace it with a ``template_name`` attribute - if we
want the default, we don't do anything; otherwise, we can subclass the view and
just set ``template_name`` to whatever we want [#]_.

.. [#] Even better, we can provide a ``get_template_name`` method which we can, for example, override to switch templates based on an attribute of the request.

Examples speak louder than words in some cases; here's an example of overriding
the class-based views equivalent of ``object_detail`` in the new system,
a class called ``DateDetailView``

.. sourcecode:: python

    import datetime
    from django.views.generic.dates import DateDetailView

    class MyDateDetailView(DateDetailView):

        # Use our article template for everything
        template_name = "advent/article.html"

        # Always use the current year if they're not authenticated
        def get_year(self):
            if not self.request.is_authenticated():
                return datetime.datetime.utcnow().year
            else:
                return super(MyDateDetailView, self).get_year()

You can see that it's very easy to provide arbitrary new code paths at key
decision points without copying-and-pasting the old function definition. In
fact, class-based views make code reuse a very easy thing; here's the actual
definition of that ``DateDetailView`` in the Django 1.3 source code

.. sourcecode:: python

    class DateDetailView(SingleObjectTemplateResponseMixin, BaseDateDetailView):
        """
        Detail view of a single object on a single date; this differs from the
        standard DetailView by accepting a year/month/day in the URL.
        """
        template_name_suffix = '_detail'

All of the generic, class-based views inside Django 1.3 are written like this,
as a series of reuseable components. Of the two classes inherited from here,
``SingleObjectTemplateResponseMixin`` brings in code which deals with providing
a queryset to get items from, and which calls a ``get_object`` method to get an
object to render; ``BaseDateDetailView`` brings in a ``get_object`` method,
along with a variety of supporting methods (like ``get_year`` above).

The aim is to not only make our life as core maintainers easier, but to
encourage people to use and expand upon this core functionality of Django.
We've been shipping similar code for years with the generic views, but their
eventual inflexibility (there's only so many options one can add in the
function signature before it becomes ridiculous) meant that a lot of developers
simply ignored them.

There's also no need to use these specific views, like ``DateDetailView`` [#]_.
There are basic ``View`` and ``TemplateView`` methods, which provide
method-based dispatch (so you can write GET, POST and DELETE as separate
methods) and standard template-rendering code respectively (a lot of views need
only inherit from ``TemplateView`` and define ``template_name`` and
``get_context_data``). The request and positional/keyword URL arguments are
also available on ``self`` [#]_, so there's no need to pass them around.

.. [#] In fact, all the new class-based generic views come in a BaseXView and an XView variant - the Base version is if you want to use some of the methods without that view's specific rendering logic.

.. [#] They're ``self.request``, ``self.args`` and ``self.kwargs`` respectively. You can store your own state on ``self`` if you want as well; it's perfectly threadsafe.

URLconfs
--------

Of course, there's a snag - there always is with any new way of doing things.
In this case, it's how you refer to class-based views in the URLconfs.

Previously, one referred to (function-based) views like so

.. sourcecode:: python

    urlpatterns = patterns('',
        (r'^awesome/$', 'advent.views.awesome')
    )

Now, instead of using a string, you must import the class and use it directly
instead, like so

.. sourcecode:: python

    from advent.views import AwesomeView
    urlpatterns = patterns('',
        # Note you can pass attributes in here to override them as well, rather than subclassing
        (r'^awesome/$', AwesomeView.as_view(template_name='advent/awesome2.html'))
    )

Some of you may think that this is a step backwards (particularly the
``as_view()`` bit), but there are good reasons for the decision, revolving
around thread-safety, not breaking normal Python idioms, and keeping it
relatively user-friendly. If you're interested, there was a thread with over
200 posts on django-developers; it makes good bedtime reading [#]_.

.. [#] It's also one of several threads on the topic; nothing else in recent years has caused as much debate about small implementation details, especially when there's three competing proposals which all have some merit.

Nevertheless, in the end a decision was reached, and ``as_view()`` is the
result. It's relatively easy to understand - it returns a new function which,
when called, makes a new class instance, calls ``dispatch()`` [#]_ on that
class with the request, and returns the response. There is, ironically, no use
of ``__call__`` in the final version of class-based views, but it's certainly
what inspired them in the first place.

.. [#] ``dispatch()`` is the method which takes the request (and other arguments from the URL), calls the relevant method (``get()``, ``post()``, etc.), and returns their response.

Further Reading
---------------

This article was more an introduction to the theory behind class-based views,
and why the version we're shipping is designed the way it is - to get started
using them, the Django documentation has an extensive `introductory section to
the class-based views <http://docs.djangoproject.com/en/dev/topics/class-based-views/>`_,
as well as a `reference to all of the view classes we now ship
<http://docs.djangoproject.com/en/dev/ref/class-based-views/>`_.

Class-based views are going to take some getting used to - I don't think
anyone's expecting them to be adopted overnight, and they'll probably never
replace function-based views entirely (indeed, that's not the intention; this
release comes with a few changes to shortcuts for use in function-based views
too).

Still, especially for highly similar sets of views, and third-party view code,
they're hopefully going to result in less duplication, and more flexible 
views - you'll be able to easily override tiny parts of third-party apps 
without forking their codebase. I can't wait to see what everyone does with 
them.

