:Author:
    Your Name Here!

#####################
Small Things Round-up
#####################

`assertNumQueries`
------------------

You've got a bunch of tests for your project (right?), you've put it into
production, and you're starting to get traffic.  So you do some optimizations,
maybe add some `select_related()` calls or cache a few things.  But how do you
verify that you don't accidentally screw these up in the future?  Before
Django 1.3 the answer was to cobble something together to verify that the
right number of queries were executed (I know I had 3 or 4 different
implementations of this strewn across my projects).  Now, however, there's a
solution included in the box!  A new `TestCase.assertNumQueries` method::

.. sourcecode:: python

    class MyTest(TestCase):
        def test_my_method(self):
            obj = MyObject()
            self.assertNumQueries(3, obj.method, 2)

Or, if you're on Python 2.5 you can use it as a context manager::

.. sourcecode:: python

    class MyTest(TestCase):
        def test_my_method(self):
            obj = MyObject()
            with self.assertNumQueries(3):
                obj.method(2)

Easy as pie (that's a saying, right?)!


Some ideas for various small things that would be worth a paragraph or
more:

* 1.3 deprecations?
* clearable file inputs
* no need for import * in root URLconf
* transaction context managers
* sane "current Site" API
* no more naughty words!
* Small speed ups
* Buildbots
* TemplateResponse
* assertNumQueries
