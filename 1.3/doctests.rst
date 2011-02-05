:Author:
    Alex Gaynor

########################################
We have always been at war with doctests
########################################

Ever seen your terminal overflow, after a single test failure caused thousands
of lines of output?  You were probably using doctests.  Once upon a time
Django's test suite was almost exclusively doctests, but now they have been
cleansed.  Tens of thousands of lines of tests converted.

Rewrites are evil, right?  Why would we do something so completely insane,
doctests can't be *that* bad, right?

doctests are the spawn of the devil
===================================

Yes.  They really are that evil.


They obscure errors
-------------------

By default when a doctest hits a failure it'll keep on going, rather than stop
there, like normal code does.  This results in exceptionally long error
messages that you have to scroll through, looking for the root of your issues:

.. sourcecode:: python
    """
    >>> a = get_val()
    >>> a
    3
    >>> a = a + 4
    >>> a
    7
    >>> a += 3
    >>> a
    10
    """

Given this test, you might reasonable expect a single line error message, after
all there's only really one bug: ``get_val()`` doesn't actually exist,
everything after that is just a result of this.  If you were expecting a sane
output, I'm sorry to tell you, you're wrong.  This is the insanity doctest will
give you:

.. sourcecode:: python

    **********************************************************************
    File "test.py", line 2, in test
    Failed example:
        a = get_val()
    Exception raised:
        Traceback (most recent call last):
          File "/usr/lib/python2.6/doctest.py", line 1253, in __run
            compileflags, 1) in test.globs
          File "<doctest test[0]>", line 1, in <module>
            a = get_val()
        NameError: name 'get_val' is not defined
    **********************************************************************
    File "test.py", line 3, in test
    Failed example:
        a
    Exception raised:
        Traceback (most recent call last):
          File "/usr/lib/python2.6/doctest.py", line 1253, in __run
            compileflags, 1) in test.globs
          File "<doctest test[1]>", line 1, in <module>
            a
        NameError: name 'a' is not defined
    **********************************************************************
    File "test.py", line 5, in test
    Failed example:
        a = a + 4
    Exception raised:
        Traceback (most recent call last):
          File "/usr/lib/python2.6/doctest.py", line 1253, in __run
            compileflags, 1) in test.globs
          File "<doctest test[2]>", line 1, in <module>
            a = a + 4
        NameError: name 'a' is not defined
    **********************************************************************
    File "test.py", line 6, in test
    Failed example:
        a
    Exception raised:
        Traceback (most recent call last):
          File "/usr/lib/python2.6/doctest.py", line 1253, in __run
            compileflags, 1) in test.globs
          File "<doctest test[3]>", line 1, in <module>
            a
        NameError: name 'a' is not defined
    **********************************************************************
    File "test.py", line 8, in test
    Failed example:
        a += 3
    Exception raised:
        Traceback (most recent call last):
          File "/usr/lib/python2.6/doctest.py", line 1253, in __run
            compileflags, 1) in test.globs
          File "<doctest test[4]>", line 1, in <module>
            a += 3
        NameError: name 'a' is not defined
    **********************************************************************
    File "test.py", line 9, in test
    Failed example:
        a
    Exception raised:
        Traceback (most recent call last):
          File "/usr/lib/python2.6/doctest.py", line 1253, in __run
            compileflags, 1) in test.globs
          File "<doctest test[5]>", line 1, in <module>
            a
        NameError: name 'a' is not defined
    **********************************************************************
    1 items had failures:
       6 of   6 in test
    ***Test Failed*** 6 failures.

Good luck debugging any of that.  If that wasn't bad enough, this is compounded
by the fact that doctests don't support any of the usual mechanisms for
splitting up your code.  Where with a unittest you'd have a dozen individual
tests, in a doctest you basically have to make them all part of one test,
because you don't have ``setUp()``, ``tearDown()``, or ``fixtures`` support.

They're stupid strings
----------------------

Meaning your editor doesn't syntax highlight them and tools like ``pyflakes``
or ``pylint`` can't analyze them.  That is, every tool you normally have at
your disposal for understanding your code is rendered useless.  Ever noticed a
bug while programming because your editor didn't syntax highlight a keyword
because you'd typo'd it?  I have, quite a few times actually.  Not once while
writing a doctest though.  I usually found out about those in the middle of
7000 line error messages.

They encourage weak assertions
------------------------------

Given a test like:

.. sourcecode:: python

    """
    >>> a
    8
    """

You might reasonably assume that you're testing that ``a`` has a value of
``8``.  You're not.  You're testing that ``repr(a) == "8"``.  A much weaker
check.  Compare that to:

.. sourcecode:: python

    self.assertEqual(a, 8)

Which actually does check that ``a == 8``.  This doesn't affect 99% of code;
how often do you have an object that's incorrect, but that also has a matching
``repr()``?  Probably not that often. On the other hand, if that ever happens
when you're writing a doctest you'll probably spend the next 6 weeks trying to
debug it.

doctests are a pain in the ass to write
---------------------------------------

Sure it's alluring to just write something simple like:

.. sourcecode:: pycon

    >>> max([1, 2])
    2
    >>> max([2, 1])
    1

But what if you want to do something more complicated, say open up a file?

.. sourcecode:: pycon

    >>> with open(path) as f:
    ...     contents = f.read()
    ...

Have fun lining up all those dots.

Crappy interaction with the Django test runner
----------------------------------------------

This one isn't strictly doctest's fault, but it was a good reason to switch.
When you run a normal Django unittest Django will either create a transaction
to isolate your test, or completely rebuild the database.  It does neither when
running a doctest.  To work around this and ensure a clean database state many
doctests would do ``call_command("reset")``, which deletes every table and
recreates them.  This is absurdly slow on most databases (anything but SQLite),
and completely unnecessary.  Converting these doctests into unittests and
removing these calls (since they were isolated with a transaction) resulted in
massive speedups to the Django test suite.


I don't know about you, but I was convinced after the first point.  7000 lines
of traceback for 1 line of failure sucks.  But readability, debug-ability,
speed, and tooling all from one patch?  SOLD!

Seeing the light
================

There isn't a ton to be said for the conversion itself, other than it was
exceptionally tedious, tiring, and you should go out and thank anyone who ever
killed a doctest.

doctests and you
================

Your company has a nice big test suite (it does have a test suite, right?). If
you started it a long time ago it probably has a bunch of doctests.  Should you
convert them all?  Yes.  It doesn't have to happen all at once, but in the long
run you must kill all of your doctests, preferably in the most brutal fashion
imaginable.  Every time you start to write a new test, convert a few doctests
from the file you're working in.  It won't be too long until you're killed them
all.

Is there a time and a place for doctests?
=========================================

I've been hard on doctests, I've compared them to Satan (and they didn't come
out favorably), however there is one time and place for doctests: as
documentation that happen to be executable.  When in doubt remember the golden
rule: doctests are docs first, and tests never.
