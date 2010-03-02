:Author:
	Eric Holscher

###########################
Django Testing Improvements
###########################

The testing realm in Django is mostly composed of two audiences: contributors
to Django itself (running the substantial existing suite of tests) and
application developers with their own suite of tests. Django 1.2 brings a few
improvements to the testing infrastructure aimed at making the tester's life a
little bit easier in both of these scenarios.

Multiple Databases
==================

What Changed
------------

The introduction of multiple database support touched many parts of Django, and
the testing framework is no exception.

Django Developers
-----------------

Django's tests now require a more detailed settings file in order to pass. If
you run the test suite with your old settings file, which simply defined a
single database, the MultiDB tests will fail (22 of them, to be exact). An
example of the minimum settings file needed is supplied below. As of the Django
Pycon sprint, this file is actually shipped with Django itself. So in order to
run the Django test suite on SQLite, you don't have to touch your filesystem!

.. sourcecode:: python

    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.sqlite3',
        },
        'other': {
            'ENGINE': 'django.db.backends.sqlite3',
            'TEST_NAME': 'other_db',
        }
    }

To run the Test Suite, ``cd`` into the `tests` directory of your Django
checkout, and run the following command::

    ./runtests.py --settings=test_sqlite

This will run with the supplied `test_sqlite.py` file, which is identical to
the above settings file.

App Developers
--------------

This only really affects you if your application expects to be deployed against
a multiple database environment. The Django documentation on testing has
specific topics about `Testing Master/Slave setups`_, which allows you to use
the ``TEST_MIRROR`` setting on a slave database and have it mirror the master
for tests. The Django TestCase also added the  `multidb`_ attribute, which will
flush all databases before running the test cases inside. By default, Django
only flushes the default database, as an optimization.

However, unless you are specifically writing against a multible database
environment, you should be able to safely ignore these changes.


Fail fast and fail early
========================


If you are running the test suite, it is presumably because you expect it to
pass. So when you run the suite and see that a test has failed, more often than
not, that is all the information that you need to know. If you have a long
suite, you don't want to have to wait for it to finish running before you see
the error, since that error is what you care about.

What Changed
------------

As of `Revision 11843`_, Django added a ``--failfast`` option to the test
runner. When this option is selected, Django will exit the test run once it
encounters a failure. This is useful for certain work flows, including some
forms of Test Driven Development, where you expect tests to fail often, and
don't want to run through the entire test to see what has failed.

In Django 1.1 and before, if you were to <Ctrl+C> and cancel a test run, you
would be treated with a nice ugly traceback with no information.

.. sourcecode:: python

    ............^CTraceback (most recent call last):
      File "./manage.py", line 11, in <module>
        execute_manager(settings)
      File "/Users/eric/svn-clones/django/template/__init__.py", line 285, in parse
        compiled_result = compile_func(self, token)
    KeyboardInterrupt


When you try the same trick as of `Revision 12034`_, the current test will
finish running and exit gracefully.

.. sourcecode:: sh

    .....................^C <Test run halted by Ctrl-C> .
    ----------------------------------------------------------------------
    Ran 22 tests in 1.308s

    OK

This allows you to get the same benefits from the ``--failfast`` option, even
if you forget to set it at the beginning. Once you see a test fail, you can
simply ``Ctrl+C``, and you will get your failing test output back out.


Django Developers
-----------------

This is important for developing against the Django source tree because the
Django test suite takes a long time to run. The full suite can take upwards of
10 minutes, especially for bigger databases like Oracle. Previously, if you saw
a failure you would just have to wait until the tests had finished running to
be able to see the actual error. Breaking out early would result in the loss of
the data you were after! Now you will be able to easily bail out on the first
failure, or if you're forgotten the ``--failfast`` option, a simple ``Ctrl+C``
will do it for you.


App Developers
--------------

This is important for integrating testing into your work flow. One of the main
ideas behind testing is that you should be running your tests while you are
developing, and see if you are making errors while you are writing the code.

Adding the ``--failfast`` argument to whatever mechanism you have to run your
tests after changes, will allow you to work faster when things break. You will
be alerted right away to breakages in your code, and be able to fix them.

There are a number of other features that would be nice to have in this realm
as well. Being able to run your full test suite, and then only run the failing
tests from the previous run is another optimization here. This will hopefully
make it into Django 1.3, or be realized as a stand-alone app before then.

There are also a number of efforts to integrate your Django tests with nose. So
if you haven't started writing your test suite, looking at `django-nose`_ or
`django-sane-testing`_ might make sense, as nose already has a lot of these
features built in.


Keepin’ it classy
=================

The (arguably) largest improvement to Django's testing infrastructure in 1.2 is
class-based test runners. This is a big deal in that it allows for easier
subclassing of the current test runner; anyone who wants to change how Django's
tests are run has a much easier job to do.

What Changed
------------

As of `Revision 12255`_, the default test runner for Django is now a class.
There is code that will detect if you are trying to pass in a 1.1-style
function based runner, and do the appropriate thing. However, you should try to
update custom test runners to the new style, because it will result in being
less code and more maintanable.

This is only really applicable to application developers, because if you are
running the Django test suite, you should be using the official Django test
runner.

An example
----------

In my `Pony Utils`_ project, I wanted to create my own `test runner`_, and
pre-1.2, that required me to copy 60 lines of code, only to edit a few (line
60). The `run_tests function`_ in Django is now 7 lines of code, which calls
out to other methods. These other methods are easy to subclass in your own test
runner, overriding only the parts needed to get the desired behavior.

The `diff`_ of the new and old show the changed lines of code. However, this
really should be done in the ``suite_result`` function on the test runner
class, since I only want to modify how the results of the tests are sent. This
shows a place where the design could be improved.

The early Djangonaut gets the Pony
==================================

Testing pre-release software allows users to catch bugs and find use cases that
the core developers haven't thought about -- and address them before they are
baked into a production release that must be supported. If a new feature isn't
well tested before release, Django is stuck supporting that functionality for
at least three releases going forward, and everyone gets suboptimal behavior.

When I was writing this article, I found a way that the new class based test
runner could be better used. The ``suite_result`` function of the test runner
wasn't being passed the actual suite to report the results on. The default
Django functionality here is just to report the number of errors and exit, but
I wanted to do more, and needed the suite. So I filed a `ticket`_, with my
example use case that wasn't supported, and hoped for the best.

Luckily this happened during the Pycon sprints, and Russell Keith-Magee
accepted the ticket and `patched Django`_ within a day! This case is a great
example of how using the pre-release version of Django can lead to making the
software better.

Thanks for reading
==================

I hope this article has made testing more enlightening for you, and has helped
highlight the new features in the 1.2 release. Now go out and use them to
improve the quality of your code, and Django itself!


.. _multidb: http://docs.djangoproject.com/en/dev/topics/testing/#multi-database-support
.. _Testing Master/Slave Setups: http://docs.djangoproject.com/en/dev/topics/testing/#testing-master-slave-configurations
.. _Revision 12034: http://code.djangoproject.com/changeset/12034
.. _Revision 11843: http://code.djangoproject.com/changeset/11843
.. _Revision 12255: http://code.djangoproject.com/changeset/12255
.. _django-nose: http://github.com/jbalogh/django-nose/
.. _django-sane-testing: http://devel.almad.net/trac/django-sane-testing/
.. _Pony Utils: http://github.com/ericholscher/pony_utils
.. _test runner: http://github.com/ericholscher/pony_utils/blob/master/pony_utils/django/test_runner.py
.. _run_tests function: http://code.djangoproject.com/browser/django/trunk/django/test/simple.py?rev=12375#L300
.. _diff: http://github.com/ericholscher/pony_utils/commit/50355eaab2ae42e327a06380d975fae9ab33e19b
.. _ticket: http://code.djangoproject.com/ticket/12932
.. _patched Django: http://code.djangoproject.com/changeset/12487

