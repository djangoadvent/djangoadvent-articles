:Author:
    Paul McMillan

#########################
More Testing Improvements
#########################

Please make your first paragraph a brief introduction that can be shown as
an article summary elsewhere, standalone.

Testing Tools
=============

The release of Django 1.3 introduces a number of improvements to testing. 

Unittest2
---------
http://www.voidspace.org.uk/python/articles/unittest2.shtml

better comparison failure results

cleanup

test skipping
setup/teardown
 expected failure


- skipIfDBFeature
- skipUnlessDBFeature

assertQuerysetEqual
assertNumQueries

Deprecated TestRunner
-unittest2.texttestrunner is replacement

TEST Dependencies
-----------------
ordering of databases

RequestFactory
--------------


Testing Django itself
=====================


bisection paired testing
------------------------
-14079 russelm



Here are some callouts:

.. attention::

   Caution: this is an important warning!

.. tip::

   Tip: It's important to ensure that data is never written to your slave.


Here's some sourcecode:

.. sourcecode:: python

    >>> from foo.models import Bar
    >>> Bar.objects.get(pk=1)

And a link_.

.. _link: http://foo.com/bar


You can use footnotes, too:

Blah blah here's some sample text. [#]_ And here's some more text in the
sample paragraph.

.. [#] He went only by the codename 'Jake Obyan'.


