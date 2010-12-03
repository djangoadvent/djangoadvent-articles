:Author:
    Carl Meyer

###########
Delete This
###########

..

    "Django's a web framework, not a deletion framework."  -- Malcolm
    Tredinnick, PyCon 2010

Nevertheless, we web-developers still find ourselves needing to delete things
now and again. [#]_ And it seems to rub people the wrong way when they forget
that even ForeignKeys with ``null=True`` are cascade-deleted by the Django
ORM. Especially if they happen to forget this while making, you know, "just a
quick tweak" to the production database (pro tip: don't do that).

Thanks to some `excellent patches`_ submitted by Michael Glassford and Johannes
Dollinger, we've fixed that for Django 1.3. Now you can make ForeignKeys
cascade, or not, or set null, or whatever you want them to do on delete. So be
forewarned: the next time you screw up your production database, there'll be
one less good excuse for it.

.. [#] Yeah, this quote's a little bit out of context. Malcolm was actually
   talking about bulk-deletion performance at the time, not customizing
   delete-cascade. Of course, in the process we `fixed the performance issue
   too`_.

.. _excellent patches: http://code.djangoproject.com/ticket/7539

.. _fixed the performance issue too: `Improved performance`_

What exactly is the problem here?
=================================

Let's back up. Say you've got models named ``Cheesemaker`` and ``Cheese``. And
of course ``Cheese`` has a ``ForeignKey`` to ``Cheesemaker``, because cheese
doesn't grow on trees, you know. So it looks like this::

    from django.db import models

    class Cheesemaker(models.Model):
        name = models.CharField(max_length=100)

    class Cheese(models.Model):
        name = models.CharField(max_length=100)
        maker = models.ForeignKey(Cheesemaker)

If you delete a cheesemaker, Django will automatically delete all their cheeses
as well. Seems reasonable enough: no cheesemaker, no cheese.

But let's try something else. What if we wanted to record each cheesemaker's
favorite cheese? Cheesemakers don't have to have a favorite, so we make it
nullable [#]_::

   class Cheesemaker(models.Model):
       name = models.CharField(max_length=100)
       favorite_cheese = models.ForeignKey("Cheese", null=True)

.. [#] You noticed the quotes around ``"Cheese"``? Sharp eyes. That's because
   we can't reference the Cheese model directly in the Cheesemaker definition;
   Cheese hasn't been defined yet and this is module-level code.

Great. But now what happens if we delete a cheesemaker's favorite cheese? The
cheesemaker gets deleted, and all of their cheeses with them. Probably not what
we want. Instead, we'd like to be able to tell Django: "if I delete a
cheesemaker's favorite cheese, just set their favorite_cheese to NULL/None."
And in Django 1.3, we can do exactly that::

    class Cheesemaker(models.Model):
       name = models.CharField(max_length=100)
       favorite_cheese = models.ForeignKey("Cheese", null=True,
                                           on_delete=models.SET_NULL)

Now when we delete the cheesemaker's favorite cheese, they won't be deleted
along with it (though they might get quite angry).

What are my options?
====================

Why not let the database do it?
===============================

Other benefits
==============

Improved performance
--------------------

Simpler code
------------
