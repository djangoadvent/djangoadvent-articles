:Author:
    Paul McMillan

#########################
More Testing Improvements
#########################

Django 1.3 introduces a number of improvements to testing including unittest2, several new test assertions, and the removal of doctests from Django's core test suite.

unittest2
=========

Python 2.7 introduced a bunch of cool new testing features to the standard `unittest` library. The `unittest2` library is a backport of these new features to previous versions of Python. Django 1.3 allows all Django developers to take advantage of these new features by packaging the unittest2 library as `django.utils.unittest`.

Converting existing code to use the new features is easy. Simply replace every occurance of `import unittest` with `from django.utils import unittest`. This will use the Python 2.7 unittest code if available, or the system's installed version of unittest2, falling back to Django's bundled version if necessary. 

One of the most useful changes is the new behavior of `assertEqual`.

I have highlighted a few features here, but you should definitely check out Michael Foord's excellent article on the `improvements introduced with unittest2`_.

.. _`improvements introduced with unittest2`: http://www.voidspace.org.uk/python/articles/unittest2.shtml

better comparison failure results

cleanup

test skipping
setup/teardown
 expected failure

- skipIfDBFeature
- skipUnlessDBFeature

assertQuerysetEqual
assertNumQueries

Approximate
-----------

Another class introduced that makes writing tests easier is `django.test.Approximate`. While `TestCase.assertAlmostEqual()` is often all that is necessary, the new Approximate class provides new flexibility when testing dictionary results. Approximation is often necessary when results include values which may vary slightly depending on database or operating system.

An example from the aggregation regression tests:

.. sourcecode:: python

    self.assertEqual(vals, { 
        'num_authors__sum': 10, 
        'num_authors__avg': Approximate(1.666, places=2), 
        'pages__max': 1132, 
        'price__max': Decimal("82.80") 
    }) 

We construct the expected results dictionary with an `Approximate` value in place, rather than removing and comparing the approximated values separately.

Other compatibility notes
-------------------------

Some of the test result messages have changed slightly. Code which tests specific response strings may need slight adjustments. One example of this was in test_client_regress_. This is not a recommended technique in most cases.

.. _test_client_regress: http://code.djangoproject.com/changeset?new=django%2Ftrunk%2Ftests%2Fregressiontests%2Ftest_client_regress%2Fmodels.py%4014139&old=django%2Ftrunk%2Ftests%2Fregressiontests%2Ftest_client_regress%2Fmodels.py%4014106

`DjangoTestRunner` is deprecated. `unittest2`'s `TextTestRunner` provides the same functionality (primarily handling Ctrl-C interrupts during testing).


TEST Dependencies
-----------------
ordering of databases

RequestFactory
--------------


Testing Django itself
=====================

more verbose by default
Django 1.2 defaulted to running tests with verbosity set to 0, which suppresses almost all output. Since unittest2 includes more helpful error messages when tests fail, the default verbosity when running Django's test suite has been set to 1.

doctest removal

improved runtime


bisection paired testing
------------------------
-14079 russelm



