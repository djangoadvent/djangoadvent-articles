:Author:
	Sean O'Connor

Smoothing The Curve
===================

For nearly all of the tools provided by Django, there are hooks provided to
allow for incremental modification and extension.  Very rarely do you ever need
to rewrite a significant amount of functionality within Django, simply to
change the behavior of a tool.  For example, if you want change the way a form
will look, you can change a widget from the default, create a custom field, or
just use your own HTML.  With each option, you can override a bit more
functionality but still take advantage of everything else the forms library
provides.  

You can visualize these incremental options as a curve.  At one end you have a
stock form which is simple to write but provides limited control.  At the other
end, you have a custom form class with static HTML which provides much more
control but is more complicated to create.  In the case of forms this is a
pretty smooth curve since there are options to incrementally replace all of the
functionality of the library.

Before Django 1.2 the ORM had a similar curve with one exception.  It had a bit
of a cliff at the end.  In particular, if you got to the point where you needed
to just write a custom SQL query you needed to go completely outside the ORM.
While this wasn't horrible there was functionality which one would still want
from the ORM which one had to now rebuild on their own.  With Django 1.2, a
``Model.objects.raw()`` method has been added to address this problem and
smooth out the ORM's curve.

The Old Way
===========

Before Django 1.2 if you needed to do a raw SQL query, you needed to do
something like:

.. sourcecode:: python

    from django.db import connection
    from library.models import Author
    
    cursor = connection.cursor()
    query = "SELECT * FROM library_author"
    cursor.execute(query)
    results = cursor.fetchall()
    
    authors = []
    for result in results:
        author = Author(*result)
        authors.append(author)
    
Now this isn't horrible, but we have lost access to functionality in the ORM
beyond the ability to generate SQL queries.  In particular, we've lost the
automatic transformation of the results of our query into model instances.
There are ways we could replicate the lost functionality without too much work
but we'd be reinventing the wheel.

The New Way
===========

In Django 1.2 to perform a raw SQL query, you can do something like:

.. sourcecode:: python

    from library.models import Author
    
    query = "SELECT * FROM library_author"
    authors = Author.objects.raw(query)
    
The result here, ``authors``, is a ``RawQuerySet``.  ``RawQuerySet`` is much
like ``QuerySet`` in that it is an iterable object which returns a model
instance from the result set with each iteration.  It is not like a
``QuerySet`` in that it cannot be chained.  Since the query isn't being built
programatically anymore, chaining doesn't make sense.

Similar to a raw database cursor, we can provide a collection of parameters to
the raw method and Django will safely quote/escape them.

.. sourcecode:: python

    query = "SELECT * FROM library_author WHERE first_name = %s"
    params = ('bob',)
    authors = Author.objects.raw(query, params)

So this is great! We're not reinventing the wheel, we're protected from SQL
injection attacks, and we have the model instances we want.

"But wait, there's more!"
=========================

Like most things in Django, the ``raw()`` method offers some additional
functionality to help with corner cases or particularly complex queries:

Field order independence
------------------------

``Model.objects.raw()`` doesn't care what order fields are returned in by the
query, all that matters is that the query field names match up to a field on
the model.

.. sourcecode:: python

    # All of these queries will work the same
    Author.objects.raw("SELECT * FROM library_author")
    Author.objects.raw("SELECT id, first_name, last_name FROM library_author")
    Author.objects.raw("SELECT last_name, id, first_name FROM library_author")

Annotations
-----------

If a query returns any fields which do not exist in the model class, they are
added as annotations to the model instances returned by the ``RawQueryset``.
This allows you to easily take advantage of operations or calculations which
are more efficient to perform within the database. [#]_

.. sourcecode:: python

    >>> authors = Author.objects.raw("SELECT *, age(birth_date) as age FROM library_author")
    >>> for author in authors:
    ...     print "%s is %s." % (author.first_name, author.age)
    John is 37.
    Jane is 42.
    ...

Field Mappings
--------------

If for whatever reason, your query field names cannot exactly match your model
field names, ``Model.objects.raw()`` provides a facility for mapping query
fields to model fields. [#]_

To map query fields to model fields, one simply needs to pass a dictionary
containing the translations to the ``raw()`` method.  Only fields which don't
match model fields need to have translations provided.

.. sourcecode:: python

    field_map = {'first': 'first_name', 'last': 'last_name}
    query = 'SELECT id, first_name AS first, last_name as last FROM library_author'
    authors = Author.objects.raw(query, translations=field_map)

Deferred Fields
---------------

Any fields which are expected by the model, but are not returned by the query,
are marked as `deferred
<http://docs.djangoproject.com/en/dev/ref/models/querysets/#queryset-defer>`_.
Deferred fields are only fetched when the model instance's field is accessed.
This is useful in cases where you may not be pulling data from the "real" table
for the model or when you have very large tables.  Be aware that primary keys
cannot be deferred and must be returned by all queries.  If a query doesn't
return a primary key, an ``InvalidQuery`` exception will be raised.

Limitations
===========

There are a few limitations placed on what ``raw()`` can do.  The biggest of
which is that ``raw()`` will only allow ``SELECT`` queries.  If any other type
of query is attempted via ``raw()``, an ``InvalidQuery`` exception will be
raised.  This is done partially because it doesn't make sense to return model
instances for anything other than ``SELECT`` queries but it is primarily done
as a deterrent.  Modifying data with raw SQL is very much something which
should be an absolute last resort in Django.  Accordingly we didn't want to
encourage the practice by making it any easier to do so.  If you really need to
perform raw SQL queries which are not ``SELECT`` queries, you can still get a
raw database cursor and go from here.

That's all folks
================

There you have it.  Now in Django 1.2, you can much more easily perform raw SQL
queries when you need to.  The curve has been smoothed.  Official documentation
for this new feature can be found on the `raw SQL
<http://docs.djangoproject.com/en/dev/topics/db/sql/#topics-db-sql>`_ page. [#]_


.. [#] Example heavily stolen from the `django docs <http://docs.djangoproject.com/en/dev/topics/db/sql/#django.db.models.Manager.raw>`_

.. [#] It's worth noting here that when the term "model fields" is used, it
   means the database field name that the Django ORM is expecting to exist in the
   database, not necessarily the name of the python attribute on the model class.
   If you've overridden a field name using ``db_column``, the override name is
   what the ``raw()`` method will be expecting.

.. [#] Thanks to Jacob Kaplan-Moss for finishing the ``raw()`` work where I
   left it off and to Russell Keith-Magee for contributing the code to handle
   deferred fields.
