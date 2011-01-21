:Author:
    Carl Meyer

###########
Delete This
###########

..

    "Django's a web framework, not a deletion framework."  -- Malcolm
    Tredinnick, DjangoCon 2010, hallway track [#]_

Nevertheless, we web developers still find ourselves needing to delete
things.

Once in a while, someone new to Django discovers that ForeignKeys (even with
``null=True``) are cascade-deleted by the Django ORM. Sometimes this makes them
sad. Or angry.  Especially if they happen to discover this while making, you
know, "just a quick tweak" to the production database (pro tip: don't do that).

Thanks to some `excellent patches`_ submitted by Michael Glassford and Johannes
Dollinger, we've fixed that for Django 1.3. Now you can make ForeignKeys
cascade, or not, or set null, or whatever you want them to do on delete.

So be forewarned: the next time you delete half of your production database,
there'll be one less excuse for it.

.. [#] Double points for a quote both reconstructed from memory and taken out
   of context (but used with permission). If memory serves, Malcolm was
   discussing bulk-deletion performance at the time, not customizing
   cascade-delete. Of course, in the process we've `fixed the performance
   issue, too`_.

.. _excellent patches: http://code.djangoproject.com/ticket/7539

.. _fixed the performance issue, too: `Improved performance`_

What exactly is the problem here?
=================================

Let's back up. Say you've got models named ``Cheesemaker`` and ``Cheese``. And
of course ``Cheese`` has a ``ForeignKey`` to ``Cheesemaker``, because cheeses
don't grow on trees. So it looks like this::

    from django.db import models

    class Cheesemaker(models.Model):
        name = models.CharField(max_length=100)

    class Cheese(models.Model):
        name = models.CharField(max_length=100)
        maker = models.ForeignKey(Cheesemaker)

If you delete a cheesemaker, Django will automatically delete all their cheeses
as well. Seems reasonable enough: no cheesemaker, no cheese.

But let's try something else. What if we wanted to record each cheesemaker's
favorite cheese? Cheesemakers aren't required to play favorites, so we make it
nullable::

    class Cheesemaker(models.Model):
        name = models.CharField(max_length=100)
        favorite_cheese = models.ForeignKey("Cheese", null=True)

Now what happens if we delete a cheesemaker's favorite cheese? The cheesemaker
gets deleted too, and all of their cheeses with them. Oops; that's probably not
what we want.

Instead, we'd like to be able to tell Django: "if I delete a cheesemaker's
favorite cheese, just set their favorite_cheese to NULL/None."  And in Django
1.3, we can now do exactly that::

    class Cheesemaker(models.Model):
        name = models.CharField(max_length=100)
        favorite_cheese = models.ForeignKey("Cheese", null=True,
                                           on_delete=models.SET_NULL)

Now when we delete the cheesemaker's favorite cheese, they won't be deleted
along with it (though they might get quite angry: you did move their cheese).

What are my options?
====================

`SET_NULL`_ is only one of the possible values for the new ``on_delete``
argument to a ``ForeignKey``. You can also use `CASCADE`_, `PROTECT`_,
`SET_DEFAULT`_, `SET()`_, or `DO_NOTHING`_. Let's go over each one of these.

SET_NULL
--------

We've seen this one in action: does what it says on the tin. Obviously, the
ForeignKey must also be ``null=True``; if it isn't, Django will complain at
you.

CASCADE
-------

This is the default value for ``on_delete`` to maintain backwards compatibility
with previous versions of Django, so there's probably no reason you'd ever need
to set it (though you can, if you want an explicit reminder in the code).

.. note::

   Unfortunately, backwards-compatibility means ``CASCADE`` has to remain the
   default for all ForeignKeys. In an ideal world the default for nullable
   ForeignKeys would probably be `SET_NULL`_.

PROTECT
-------

Say we want to require all cheesemakers to declare a favorite cheese. We still
need to solve the problem of someone deleting a favorite cheese, but now we
can't use `SET_NULL`_ (because ``favorite_cheese`` is no longer
nullable). Instead, we decide we want to outright prevent the deletion of any
cheese that is any cheesemaker's favorite (due to heavy pressure from the
cheesemaker's guild, no doubt). Here's how we could do that::

    class Cheesemaker(models.Model):
        name = models.CharField(max_length=100)
        favorite_cheese = models.ForeignKey("Cheese", on_delete=models.PROTECT)

Now, if we try to delete any cheese that is referenced as ``favorite_cheese``
from any cheesemaker, Django will actually raise ``ProtectedError`` (a subclass
of ``IntegrityError`` and prevent the deletion from occurring.

.. warning::

   If you use ``PROTECT`` ForeignKeys, you are responsible to clear out any
   relevant relationships before attempting a delete, and/or catch
   ``ProtectedError`` from the ``delete()`` call. If your code allows the
   ``ProtectedError`` to go uncaught, it will result in a 500 Internal Server
   Error response from your application; bad news.

SET_DEFAULT
-----------

This one's similar to `SET_NULL`_, except rather than setting the ForeignKey to
NULL when its target gets deleted, it sets it to the default value for the
ForeignKey. Just like a `SET_NULL`_ ForeignKey must be ``null=True``, a
``SET_DEFAULT`` ForeignKey must have a default value, or Django will complain.

.. note::

   In almost all cases, you'll want the default for a ForeignKey to be a
   callable that returns the "default" target instance; otherwise, you'd be
   querying for that default instance at module-import time, which can cause
   all sorts of troubles.

You guessed it: I'm going to try this one out on our poor confused
cheesemakers. We're going to specify a region for each cheesemaker; we'll say
most of our cheesemakers happen to come from western Switzerland, so we'll make
the Emmental the default region::

    def get_default_region():
        return Region.objects.get_or_create(name="Emmental")[0]

    class Region(models.Model):
        name = models.CharField(max_length=100)

    class Cheesemaker(models.Model):
        name = models.CharField(max_length=100)
        region = models.ForeignKey(Region, default=get_default_region,
                                   on_delete=models.SET_DEFAULT)

Now if we delete a Cheesemaker's region, they'll revert to Emmental.

SET()
-----

``SET()`` is the fully-flexible generic version of `SET_NULL`_ and
`SET_DEFAULT`_; you can pass any value to it (or more likely, a callable that
returns a value, for the same reasons as with `SET_DEFAULT`_), and that value
will be used as the fallback in case the target object is deleted.

For our example here, let's give favorite-cheeses a rest, and add a new twist:
cheesemakers can have site logins. Since we're using ``contrib.auth`` for
authentication, that means a OneToOneField to
``contrib.auth.models.User``.

Easy enough -- but wait. By now we're well attuned to the risks of the default
cascade deletion; if somebody should happen to delete a User, do we really want
that cheesemaker and all their cheeses to disappear into the ether? I dare say
we do not::

    from django.contrib.auth.models import User

    def get_sentinel_user():
        return User.objects.get_or_create(username="deleted")[0]

    class Cheesemaker(models.Model):
        name = models.CharField(max_length=100)
        user = models.OneToOneField(User,
                                    on_delete=models.SET(get_sentinel_user))

Now if we delete a cheesemaker's user, that cheesemaker will be re-associated
with a special ``User`` object with the username "deleted".

(Yes, ``on_delete`` works with ``OneToOneField`` as well as ``ForeignKey``.)

DO_NOTHING
----------

You may be wondering why Django implements all of this at the ORM layer, when
any SQL database worth its salt already supports ON DELETE clauses in table
definitions. The problem is, Django's ORM has to support a variety of database
backends, including some (MySQL ISAM) that don't support referential integrity
or cascade at all. Implementing cascade behaviors at the ORM level allows
Django code using ``on_delete`` to be portable to these databases, and also
allows additional flexibility (such as the `SET()`_ and `Write your own`_
options).

But all is not lost for the SQL purists among us! If you want to leave
cascade-handling entirely in the hands of your database, just use the
``DO_NOTHING`` option with your ForeignKeys and Django won't do any cascading
at all. This means it's your responsibility to ensure that your database tables
are created with the appropriate ``ON DELETE`` clauses, to avoid
``IntegrityError`` when you try to delete referenced objects.

Let's rewrite our original ``Cheese`` model. We still want deletion of a
cheesemaker to cascade and delete all their cheeses, but now we want the
database to handle it (I'll assume we're using `PostgreSQL`_)::

    class Cheese(models.Model):
        name = models.CharField(max_length=100)
        maker = models.ForeignKey(Cheesemaker, on_delete=models.DO_NOTHING)

With just this change, deleting a cheesemaker will cause an ``IntegrityError``:
we've asked Django not to cascade, but we haven't yet told Postgres to do
it.

We could connect to our database shell and manually alter the table schema to
add the ``ON DELETE`` clause, but then we'd have to remember to do that every
time we ``syncdb`` a fresh database: yuck. Instead, let's make it automatic by
adding some `initial SQL`_ in the ``sql/cheese.postgresql_psycopg2.sql`` file
in our app (presuming our app is named "cheese" as well)::

    ALTER TABLE "cheese_cheese"
        DROP CONSTRAINT "cheese_cheese_maker_id_fkey";

    ALTER TABLE "cheese_cheese"
        ADD CONSTRAINT "cheese_cheese_maker_id_fkey"
            FOREIGN KEY ("maker_id")
            REFERENCES "cheese_cheesemaker" ("id")
                ON DELETE CASCADE
                DEFERRABLE INITIALLY DEFERRED;

(In order to know the name of the constraint to drop and recreate, and the full
syntax for recreating it, I just checked the table schema in the Postgres
shell. If you're planning to use the ``DO_NOTHING`` option, you should probably
already be on good terms with your particular database and SQL syntax.)

If we drop our database and re-sync it with this added initial SQL, Postgres
will now handle the cascade deletions from cheesemaker to cheese.

.. note::

   If you are using a migrations framework such as `South`_, you could make
   this table modification in a migration rather than using initial SQL.

.. _PostgreSQL: http://www.postgresql.org
.. _initial SQL: http://docs.djangoproject.com/en/dev/howto/initial-data/#providing-initial-sql-data
.. _South: http://south.aeracode.org

Write your own
--------------

This isn't officially an option (it's not documented), but if you peruse the
source code for all of the above ``on_delete`` options, you'll notice that they
are just functions which share a common signature. With a bit of examination of
the built-in functions, you could write your own custom function and pass it to
``on_delete`` to define just about any on-delete behavior you can dream up.

.. warning::

   There's a reason this capability isn't documented; we want to give the
   argument signature for these ``on_delete`` functions a chance to shake out
   before it's set in stone. So as of now there is no backwards compatibility
   guarantee for this API: if you write a custom ``on_delete`` function, future
   Django versions might break it.

Side benefits
=============

Improved performance
--------------------

One nice side-effect of the new cascade-deletion code is that bulk-deletion of
objects referenced by ForeignKeys is much more efficient than it used to
be. Previously, relationships were followed separately and a separate query
performed on the related table for each individual object to be deleted. Now,
relationships are followed per-model, and only one bulk query is performed on
each related table.

For example, in Django 1.2 if you had 100 cheesemakers in your database and
called ``Cheesemaker.objects.all().delete()``, Django would do 100 separate
queries on the ``Cheese`` table to look for cheeses related to each one of
those cheesemakers. In Django 1.3, it will do a single bulk query on the cheese
table; and similarly on down the chain of additional relationships.

Clearer code
------------

Despite the added functionality, the new deletion code is about 50 lines
shorter, easier to follow, and easier to modify and extend. If you've got a pet
wishlist feature related to deletion in the Django ORM, there's never been a
better time to investigate it and put together a patch.

The takeaway
============

Django may not be a deletion framework, but deleting stuff in Django 1.3 is
more flexible, faster, and all around less likely to make you a sad
panda. Enjoy!
