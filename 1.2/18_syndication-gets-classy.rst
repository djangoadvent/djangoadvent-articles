:Author:
	Rob Hudson

#######################
Syndication Gets Classy
#######################

Since it's original release [#]_ Django has included support for generating RSS
feeds for content.  RSS, often expanded as "Really Simple Syndication", is an
XML format used to publish frequently updated content in a standardized format.
Similarly, Django also includes support for generating the Atom Syndication
Format [#]_, which is also a standardized format for web feeds that offers a few
`expanded capabilities`_ over RSS.

.. [#] As of `revision 3`_, from July 2005, close to 5 years ago.
.. [#] Atom feeds were added in `revision 1194`_, from November 2005.

Prior to Django 1.2
===================

While Django's syndication framework offered a very easy way to generate valid
XML, often with just a few lines of Python, it had a number of reported issues:

* Feeds had limited flexibility in the various ways developers wanted to hook
  them into `urls.py`.  For example, if your content is tagged and has HTML views
  at urls such as `/tag/django/`, it was not possible to append an `rss/`, for
  example, to the url to have `/tag/django/rss/` -- the URL had to adopt the
  pattern of `/rss/tag/django/`.  This is because the URL was constructed in
  such a way that it pattern matches a prefix plus the rest of the url:

  .. sourcecode:: python

      (r'^rss/(?P<url>.*)/$', ...),

* For more complex url patterns, as above, the URL parsing was up to the
  developer as the feed class simply split the URL based on the remaining URL
  string.  For example, the above URL regular expression would match both
  `/rss/tag/django/` and `/rss/tag/django/other/junk/`, and it was up to the
  developer to catch this and deal with this in the ``Feed`` class. The problem
  here is it leads to URL parsing being in more than one place (i.e.  somewhere
  other than the `urls.py`) and can easily be overlooked.

* There was a little wart where a content item's title and description had to be
  defined in a Django template rather than in the feed class, whereas all other
  feed attributes could be defined in Python. Often these templates consisted of
  `{{ obj.title }}` and/or `{{ obj.description }}`, which made no sense to drop
  into template rendering.

* Plus various other smaller bugs and warts and inconsistencies.

All of the above issues have been fixed in Django 1.2.  In total, `changeset
12338`_ closed 10 tickets against the syndication framework.

Class-based Views
=================

Besides the above fixes, the change brought with it the introduction of
class-based views.  Previously, while the developer did create a Feed class, it
was called via function-based view (``django.contrib.syndication.views.feed``),
which split the URL to look for the key in the supplied `feed_dict`: 

.. sourcecode:: python

    # Django 1.1 and prior
    from django.conf.urls.defaults import *
    from myproject.feeds import LatestEntries, LatestEntriesByCategory

    feeds = {
        'latest': LatestEntries,
        'categories': LatestEntriesByCategory,
    }

    urlpatterns = patterns('',
        # ...
        (r'^feeds/(?P<url>.*)/$', 'django.contrib.syndication.views.feed',
            {'feed_dict': feeds}),
    )

As of Django 1.2, the Feed subclass instance is itself a view which results in a
cleaner url pattern and removes the need for a feed dictionary to be defined.
It also removes the need to wildcard match the remaining URL, bringing the URL
pattern matching back into `urls.py` and out of the Feed class:

.. sourcecode:: python

    from django.conf.urls.defaults import *
    from myproject.feeds import LatestEntries, LatestEntriesByCategory

    urlpatterns = patterns('',
        # ...
        (r'^feeds/latest/$', LatestEntries()),
        (r'^feeds/categories/(?P<category_id>\d+)/$', LatestEntriesByCategory()),
    )

If you wanted to flip the URL around, as in the tag example above, you can now
put the named pattern wherever you'd like:

.. sourcecode:: python

    urlpatterns = patterns('',
        # ...
        (r'^/categories/(?P<category_id>\d+)/feed/$', LatestEntriesByCategory()),
    )

The ``Feed`` class gets a `category_id` passed to the ``get_object`` method of
the class.  The ``get_object`` method exists for feeds that publish different
data given different URL parameters.  The ``items`` method then takes the object
returned by ``get_object`` and returns a list of objects to publish.  In the
example below, this is the last 30 entries in the given category:

.. sourcecode:: python

    from django.contrib.syndication.views import FeedDoesNotExist
    from django.shortcuts import get_object_or_404

    class LatestEntriesByCategory(Feed):

        def get_object(self, request, category_id):
            return get_object_or_404(Category, pk=category_id)

        def items(self, obj):
            return Entry.objects.filter(category=obj).order_by('-post_date')[:30]

        def title(self, obj):
            return "Entries for category %s" % (obj.category,)

        def link(self, obj):
            return obj.get_absolute_url()

        def description(self, obj):
            return "Recent entries in category %s" % (obj.category,)

For more details on each method that can be provided, see the `feed class
reference`_.

Using the New Syndication Framework
===================================

The function-based syndication feeds will be around until 1.4, but if you'd like
to take advantage of some of the fixes class-based syndication feeds bring,
upgrading is pretty straight-forward and the `Django 1.2 release notes`_ are
probably the best source for explaining how to upgrade.

If you're not already using feeds, the `syndication feed framework docs`_ offer
all you need to get started, including the aforementioned `feed class
reference`_ for all the methods that are available to override.

.. _`expanded capabilities`: http://en.wikipedia.org/wiki/Atom_(standard)#Atom_compared_to_RSS_2.0
.. _`revision 3`: http://code.djangoproject.com/browser/django/trunk/django/core/rss.py?rev=3
.. _`revision 1194`: http://code.djangoproject.com/changeset/1194/django/trunk/django
.. _`changeset 12338`: http://code.djangoproject.com/changeset/12338
.. _`Django 1.2 release notes`: http://docs.djangoproject.com/en/dev/releases/1.2/#feed-in-django-contrib-syndication-feeds
.. _`syndication feed framework docs`: http://docs.djangoproject.com/en/dev/ref/contrib/syndication/
.. _`feed class reference`: http://docs.djangoproject.com/en/dev/ref/contrib/syndication/#feed-class-reference
