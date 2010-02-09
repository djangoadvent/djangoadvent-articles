:Author:
	Alex Gaynor


#######################################
Multiple Database Support in Django 1.2
#######################################

Since Django's creation there's been an implicit restriction to a single
database (this is as systemic as things like the ``DATABASE_*`` family of
settings), and for almost as long support for multiple databases has been
requested.  This summer, as a part of the `Google Summer of Code`_, multiple
database support was implemented, and as a part of the 1.2 process it was
merged into trunk.  These changes involve a ton of changes to the internals,
and a few well placed extensions to the existing public API.

The Multi-DB Public API
=======================

The most obvious change to Django is that instead of defining your database
settings like:

.. sourcecode:: python

    DATABASE_ENGINE = "psycopg2"
    DATABASE_NAME = "my_big_project"
    DATABASE_USER = "mario"
    DATABASE_PASSWORD = "princess_peach"

You instead write something like:

.. sourcecode:: python

    DATABASES = {
        "default": {
            "ENGINE": "django.db.backends.psycopg2",
            "NAME": "my_big_project",
            "USER": "mario",
            "PASSWORD": "princess_peach",
        },
        "credentials": {
            "ENGINE": "django.db.backends.oracle",
            "NAME": "users",
        }
    }

All projects will eventually need to be migrated these new settings, although
the old style will continue to work until Django 1.4.  The only key with
special significance in the ``DATABASES`` dictionary is ``"default"`` -- if
you're using databases you need to define a ``"default"`` database.

Now that you've told Django about all your databases you need a way to tell
Django when to use them.  The first addition in this arena is the ``using()``
method on ``QuerySets``.  ``using()`` takes a single parameter, a database
alias (alises are the keys of the ``DATABASES`` dictionary), and it binds the
``QuerySet`` to that database.  Like every other ``QuerySet`` method it can be
chained at will (like the ``order_by()`` method calling it a second time it
overides the first call).  This basically gives you complete control to define
where models are read from (and if you get tricky with ``create()``,
``delete()`` and ``update()`` even the ability to control writes):
    
.. sourcecode:: pycon

    >>> User.objects.filter(username__startswith="admin").using("credentials")

In addition to this new ``QuerySet`` method ``delete()`` and ``save()`` on
models takes a new ``using`` parameter, which is, again, a database alias.

There is also a new method on ``Managers``, ``db_manager()``, this again takes
a database alias and what it does is very similar to ``using()``.  The key
difference is that instead of returning a ``QuerySet`` it returns a new
``Manager``.  The use case for this is being able to chain it with methods on
``Managers`` that don't return a ``QuerySet``, such as ``create_user()`` on the
``UserManager``.

Database Routers
================

By using all of these methods you can implement any sort of multiple database
system you want, master-slave replication, partitioning, sharding, or anything
else.  However, it wouldn't necessarily be convenient.  You'd have to litter
your codebase with calls to ``using()``, and that doesn't play nicely with
Django's reusable application philosophy (it was for this reason that a
``using`` option was removed from the ``Meta`` class in Django models).  In
addition there are some places where you don't have an explicit ``QuerySet``,
for example ``my_obj.user`` will create a ``QuerySet`` behind the scenes to
access the ``User`` model, but you don't have any place to call ``using()``
there.  For these reason the concept of a "database router" was introduced.
Database routers take whatever is available about the query you want to
perform, and they can say what database it should go to.  Database routers are
setup in your settings file with a new ``DATABASE_ROUTERS`` setting, which is a
list of database routers:

.. sourcecode:: python

    DATABASE_ROUTERS = [
        "path.to.AuthRouter",
        "path.to.MasterSlaveRouter",
    ]

``DATABASE_ROUTERS`` is a list because at any stage a router can return
``None`` and then Django will fall back to the next router in the list.  These
routers can define a few different methods:

* ``db_for_read``
* ``db_for_write``
* ``allow_relation``
* ``allow_syncdb``

The first two are fairly self-explanitory, they return the alias that that
query should be performed against.  The ``allow_relation`` method exists to
provide a sanity check.  Django doesn't want to let you assign cross-database
relations if it's going to fail.  Therefore when you're trying to create a
relationship between two models on different databases (for either a
``ForeignKey`` or ``ManyToManyField``) Django will call this method so that you
can provide the appropriate validation.  The final method, ``allow_syncdb``,
provides a way for you to let Django know which models should be sync'd (and
therefore available) on which database.

As always the Django documentation on `multiple databases`_ provides great
examples of how to use all these things, including examples of how they're
used, and how to get started implementing some common patterns with database
routers.  The addition of multiple databases should provide a tremendous boon
for the Django community, allowing Django to be used in yet more enviroments.

.. _`Google Summer of Code`: http://code.google.com/soc/
.. _`multiple databases`: http://docs.djangoproject.com/en/dev/topics/db/multi-db/
