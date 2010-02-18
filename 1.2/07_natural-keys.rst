:Author:
	Russell Keith-Magee

############
Natural Keys
############

One of the features introduced in Django 1.2 is really little more
than a fix for a long standing bug. Natural Keys were introduced as a
way to solve a very specific problem with fixture loading.

However, there's a little bit more to this feature than a simple
bugfix. If you look outside the box, Natural Keys are a handy utility
for simplifying the way you access certain types of models.

In order to understand when natural keys are useful, it helps to know
little history.

In the beginning ...
====================

When Django's test framework was first introduced, it didn't include any
support for fixtures. If you wanted a test case to have data, you used
the ORM to define those data in the ``setUp()`` method of each
``TestCase``.

This worked, but it wasn't especially easy to use. Writing large
collections of test data was needlessly complex. There was no way to
create test data using a running Django project, and then dump that
data in a way that test framework could use it. Reusing test data
between test cases required the creation of libraries of code that set
up certain test objects.

So - a really simple test case might look something like:

.. sourcecode:: python

    from django.test import TestCase
    from library.models import Book, Author
    from library.utils import is_good_book

    class MyTestCase(TestCase):
        def setUp(self):
            self.a1 = Author()
            self.a1.first_name = 'Douglas'
            self.a1.last_name = 'Douglas'
            self.a1.save()

            self.b1 = Book(title='Mostly Harmless')
            self.b1.author = self.a1
            self.b1.save()

        def test_stuff(self):
            self.assertTrue(is_good_book(self.b1), True)

Functional? Yes. Pretty? Not especially.

Let there be fixtures!
======================

Fixture support for Django tests arrived in January 2007. There wasn't anything
especially revolutionary about Django's test fixtures -- Django's serialization
framework had been part of Django for a while.  Test fixtures just leveraged
that serialization support in a new and interesting way.

With the introduction of fixtures, you could put all the data definition into a
separate file, in a format that was suited to data definition -- JSON, XML,
YAML, or any other format for which a serializer existed.  As a result, the
test code becomes a lot clearer:

.. sourcecode:: python

    from django.test import TestCase
    from library.models import Book, Author
    from library.utils import is_good_book

    class MyTestCase(TestCase):
        fixtures = ['test_library.json']

        def test_stuff(self):
            b1 = Book.objects.get(name="Mostly Harmless")
            self.assertTrue(is_good_book(b1), True)

The data for the test is then defined in a separate test fixture file:

.. sourcecode:: javascript

    [
        {
            "id": 1,
            "model": "library.author",
            "fields": {
                "first_name": "Douglas",
                "last_name": "Adams"
            }
        },
        {
            "id": 1,
            "model": "library.book",
            "fields": {
                "title": "Mostly Harmless",
                "author": 1
            }
        }
    ]

This test fixture can be reused between tests without the need to
create test data libraries. The production of test fixtures can also
be semi-automated -- if you have a running project, you can use that
project to produce test data, and then use the ``dumpdata`` management
command to produce test data that can be used as a test fixture.

Houston, we have a problem
==========================

The test fixture format is an improvement over programmatic definition
of test data, but it isn't without flaws. For example, the test
fixture uses primary keys to reference other objects -- you define the
author of a book by specifying ``"author" : 1`` in your fixture.

This isn't an especially natural way of referencing objects -- but,
test fixture production can be automated, so unless you are manually
writing fixtures, this inconvenience doesn't really affect you.

However, the problem is slightly more than an inconvenience for
certain types of data.

The post_syncdb handler
-----------------------

Django provides a lot of runtime metadata about the objects under it's
control. The contrib.ContentTypes app provides a unique identifier for
every data type that is registered as a Django object. The
contrib.Auth app automatically creates access permissions for every
model that is defined. All this meta data is stored in normal Django
models; these models are populated using a ``post_syncdb`` handler.

Whenever you run ``./manage.py syncdb``, Django emits a
``post_syncdb`` signal. By registering a listener for this signal, it
is possible for your application to to execute code in response to a
``syncdb``. In the case of contenttypes, this means creating a new
content type entry; in the case of the auth framework, it means
creating permissions.

So what's the problem?
----------------------

The consequence of this signal-listening is that new objects will be
produced in the database as the result of a call to ``syncdb``. The
handler is told the names of the new models that have been created,
and creates content types, permissions, and other database objects to
reflect those additions.

But what are the primary keys of these new objects?

The primary keys that are allocated are determined at runtime, and
can't be predicted. The keys allocated during normal execution aren't
necessarily the primary keys that will be allocated on a test database
during test execution. The keys allocated on my database won't
necessarily match those allocated on your database.

So, if you have a model that references content types -- for example,
if you have an object that uses generic foreign keys -- then you can't
create a fixture that will consistently reproduce that object. There is
literally no primary key value you can use that will be guaranteed to
uniquely identify a content type (or any other dynamically created
data object for that matter).

This problem was the nub of `ticket #7052`_. It wasn't just a problem for
content types and permissions, either -- although these two models in
particular were the most frequently reported manifestations of the
problem. Any model that had dynamically defined data would be prone to
the same issues.

Ironically, this problem didn't actually exist in the pre-fixture
world. When you programatically define a fixture, you aren't limited
to using primary keys to reference objects (although you could if you
wanted to). You could use any attribute or combination of attributes
that uniquely reference an object.

What we really need is a way to programmatically look up a dynamic
content type in a fixture instead of hard coding a primary key value.

Natural Keys to the rescue!
===========================

This is what natural keys are for. Instead of using primary key
values to reference an object, you can use a natural key. So, if your
fixture previously had a primary key reference for a content type:

.. sourcecode:: javascript

    "content_type": 37

... you can replace that primary key value with a natural key:

.. sourcecode:: javascript

    "content_type": ["library", "book"]

The natural key itself is just a list of data values that can be used
to uniquely identify an object in the database. In the case of a
content type, the natural key is composed out of the ``app_label`` and
``model`` that forms the content type record.

This natural key will be resolved into a primary key value when the
fixture is loaded. The content type model itself provides the method
that defines how to resolve the natural key reference into an actual
database object.

Defining a natural key
----------------------

So, how do you declare that your model can be loaded using natural
keys? Easy -- you define a ``get_by_natural_key()`` method on the
default manager for the model. This ``get_by_natural_key()`` method
accepts a list of arguments; those arguments are the elements of the
list that are defined in the fixture. In the case of content types,
the ``get_by_natural_key()`` method looks something like this:

.. sourcecode:: python

    from django.db import models

    class ContentTypeManager(models.Manager):
        def get_by_natural_key(self, app_label, model):
            return self.get(app_label=app_label, model=model)

That's all there is to it. The ``get_by_natural_key()`` method is just
a method that does a specific model lookup, provided under a known
name so the serializers know how to find it.

Producing a natural key
-----------------------

Well... there is a little more. If all you want to do is load natural
keys, then all you need is the ``get_by_natural_key()`` method.
However, if you're going to use ``dumpdata`` to produce fixtures, it
would be nice to have those fixtures produced with natural keys in
place of primary key references.

To close the loop, you need to define one more method, this time on the
model itself. If your model defines a ``natural_key()`` method, Django
knows that references to objects of this type can be output as natural
key references. The ``natural_key()`` method accepts no arguments; it
returns the list of values that can be used to uniquely identify the object.

For content types, the definition of ``natural_key()`` looks like this:

.. sourcecode:: python

    from django.db import models

    class ContentType(models.Model):
        # ...
        def natural_key(self):
            return [self.app_label, self.model]

By default, Django doesn't use natural keys when it dumps fixtures. If
you want a fixture to use natural keys, you must specify the
``--natural`` option when you call ``dumpdata`` (or the
``use_natural_keys=True`` argument if you are programatically calling
the serializers).

For example, the following would dump all the objects
defined in the library application in JSON format, using 2 character
indents, using natural keys whenever they are available:

.. sourcecode:: sh

    ./manage.py dumpdata --format=json --indent=2 --natural library

Some caveats and suggestions
----------------------------

Any object can define a natural key by providing the
``get_by_natural_key()`` and ``natural_key()`` methods. However,
that natural key must be composed out of one or more attributes that
are guaranteed to be unique in the database.

The easiest way to ensure this uniqueness is to define attributes to
be ``unique_together`` in the ``Meta`` options of the model. This
enforces uniqueness at the database level; it also adds an index to
the database which will speed up the retrieval of objects by natural
key.

Uniqueness doesn't need to be enforced at the database level, but the
attributes you use in a natural key must be unique in practice. If you
choose not to define the attributes as ``unique_together``, you might
find it beneficial to add indexes to the individual columns (or define a
multi-column index). This will speed up the natural key lookups.

When using natural keys in a fixture, you need to be careful of the
order in which you define fixture objects. Natural keys are resolved
by performing a lookup on the database -- which means you can't
reference a natural key in a fixture before the object it references
has been defined.

This means that the object that is referenced must already exist in
the database before you load the fixture. If you call ``dumpdata``,
Django will dump objects in an order that contains no ambiguity.
However, if you're manually defining fixtures, you need to be careful
to ensure that the object order contains no forward natural key
references.

Back to the books...
====================

So, natural keys can be used to avoid a specific problem with test
fixtures. Is that all they are good for? In short -- no.

What is a natural key really? A natural key is a composite database
key that can be used to refer to a specific object in a database.
Although dynamic lookups in a fixture are one use for a natural key,
they can also be used in day-to-day code, whenever you need to refer
to an object in a 'natural' way.

Returning to our Author example -- an obvious natural key for an
author is the first name and last name of the author. By adding the
two natural key methods to our Author model:

.. sourcecode:: python

    from django.db import models

    class AuthorManager(models.Manager):
        def get_by_natural_key(self, first_name, last_name)
            return self.get(first_name=first_name, last_name=last_name)

    class Author(models.Model):
        objects = AuthorManager()

        first_name = models.CharField(max_length=50)
        last_name = models.CharField(max_length=50)

        def __unicode__(self):
            return u'%s %s' % (self.first_name, self.last_name)

        def natural_key(self):
            return [self.first_name, self.last_name]

...the lookup of an author:

.. sourcecode:: pycon

    >>> Author.objects.get(first_name='Douglas', last_name='Adams')

...can be replaced with a natural key lookup:

.. sourcecode:: pycon

    >>> Author.objects.get_by_natural_key('Douglas', 'Adams')

Errr... So what?
----------------

Ok -- a 6 character gain isn't really a big advantage. The elegance of
using a natural key for object lookup only becomes apparent when you
have a more complex natural key.

Consider the case of Permissions. The natural key for a permission is
composed out of the code name for the permission, and the natural key
for the content type that the permission relates to. So, the following
permission lookup:

.. sourcecode:: pycon

    >>> Permission.objects.get(code_name='add', content_type=ContentType.objects.get(app_label='library',model='book'))

...can be replaced by the much more elegant call:

.. sourcecode:: pycon

    >>> Permission.objects.get_by_natural_key('add', 'library', 'book')

Taking this a step further, natural keys don't need to map directly
onto database attributes at all. Natural keys can be composed out of
any unique value that can be converted into a unique query for an
object instance. So - if you can pass in a value (or list of values),
and parse those values in some meaningful way into a unique object
lookup, you can use those values as a natural key.

Conclusion
==========

Natural keys may not be as complex as multiple database support or
model validation, but it is a really important improvement introduced
by Django 1.2. The bug that is fixed by introducing natural keys is a
long standing problem that has caused more than one headache. There
are also benefits to be had outside of the test framework.

So - time to upgrade and enjoy your natural keys!


.. _`ticket #7052`: http://code.djangoproject.com/ticket/7052
