###################
It's Log, Log, Log!
###################


A small site is easy to manage. If you only get a few hundred hits a
day, and your site isn't especially complicated, it's easy to keep a
handle on what is going on.

However, the larger your site gets, the more important it is to get
good diagnostic information. After all, you can't improve that which
you don't measure, and you can't fix that which you can't diagnose.

What's old?
===========

Up until Django 1.2, Django's error reporting mechanisms were fairly
primitive. Any HTTP 500 errors were mailed to the site admin -- and
that was it.

This sort of error reporting really doesn't scale well.  Firstly, all
your errors get put into a single bucket -- your email inbox. Sure,
you could set up mail filters to process incoming errors, but these
can be painful to set up, and ultimately just give you more boxes, not
any real insight.

Using email as a raw reporting mechanism also leaves you victim
of the limitations of SMTP as a protocol. Any mail server outage means
you could potentially lose error reports. If a high profile bug
accidentally finds its way into production, you could find your inbox
flooded. If you use a cloud-based email provider such as GMail, you
can find that your incoming email rate is throttled, and email reports
for a bug you fix on Monday are still being delivered days later --
and that reports for errors that are happening *right now* aren't
getting delivered at all.

If you wanted to change the way Django handled 500 errors, you could
write a custom exception handler, but Django didn't provide any
alternatives or examples out of the box.

And if you wanted diagnostic information, rather than just information
on errors, you were out of luck -- again, time to roll your own
solution.

To be sure, there are ways around these problems, but they all
required custom solutions -- there wasn't anything out of the box.

What's new?
===========

Django 1.3 aims to fix this, making it trivial to get all sorts of
diagnostic information, in whatever form you need it to be in. To do
this, we haven't built anything new. We've just made it easier to use
one of Python's built in -- and often overlooked -- batteries:
Logging.

How logging works
-----------------

The documentation for Python's logging framework makes logging seem a
lot more complicated than it actually is. At its core, logging is
actually quite simple.

User space code decides it needs to log a message. This message can be
given a severity, indicating its importance, ranging from simple
debug information, to critical errors that require urgent attention.
The user writes the message to a logger, along with any metadata that
might be useful, such as a stack trace or local variables.

The logger passes the message to a handler. The handler inspects the
log message, and works out what to do with it -- write to a file, post
to a network socket, or post it as an email. The decision on which
handler to use is based on the severity of the message -- for example,
low priority debug messages may be discarded entirely.

The handler can also take into account the specific logger that
produced a message. A single project may have multiple loggers -- each
logger represents a bucket into which log messages can be directed.

Along the way, the handler or the logger can optionally define the use
of a filter. Filters provide a fine-grained way to determine whether a
message will be handled, and to modify a message en route.

Finally, the handler needs to output the logging message. The format
of the output can be configured using a Formatter. This formatter
describes exactly what information will be output by the handler, and
the format. For example, will the log message include a date? If so,
in what format will it be provided? Will it be written before or after
the error code?

And that's it! And even with all these parts, your interaction with
logging in user-space code will usually be little more than obtaining
and logger and using it

.. sourcecode:: python

    import logging

    logger = logging.getlogger('myapp.stuff')

    def my_view(request):
        logger.info("Do some stuff in my view")
        try:
            # now actually so something
        except:
            logger.error("We've had a problem")
        else:
            logger.info("Everything is groovy")


Unleash the power!
------------------

The real power of Python's logging framework comes in the various ways
that these pieces can be used.

You can create as many loggers as you want. Every logger has a name;
this name ensures that a single logger can be shared between different
code modules, or that a single code module can have multiple loggers.

Loggers are also hierarchical. The name used for loggers is a dotted
path -- e.g., ``django.db.backends``. This dotted path is used to
indicate a hierarchy; the ``django.db`` logger is a parent of the
``django.db.backends`` logger; the ``django`` logger is a parent of
the ``django.db`` logger; and there is a root logger that is the
parent of the ``django`` logger. This hierarchy can then be used as
part of the handling process. A logger can be configured to (or not
to) propagate it's errors to it's parent. This provides a way to
funnel all errors of a particular type into a common handler.

You can direct messages from a single logger to multiple handlers. For
example, you could decide to continue to email critical errors to
admin, but log all messages -- critical errors or otherwise -- to a
file. This means your reporting mechanisms don't have to be
all-or-nothing affairs; you can handle messages in as many ways as you
need.

The method of logging (file, email, etc) is decoupled from the type of
message that is being logged. So, you can write a generic email
logging handler, and plug in that handler whenever you feel that email
notification is appropriate.

Filters give you the power to transform, as well as simply filter
logging messages. The obvious case for a filter is to do processing
like "only log debug level messages produced by an admin". However,
you can also use filters to *transform* messages on their way to the
handler. For example, Django logs 4XX series messages with a severity
of INFO. If you want to make a special case of HTTP404 responses and
log them as an error, you just write and install a filter that
promotes those messages

.. sourcecode:: python

    class Error404Filter(logging.Filter):
        def filter(self, record):
            if record.status_code == 404:
                record.levelno = logging.ERROR
                record.levelname = 'ERROR'
            return True


Configuring logging
-------------------

None of this is new -- Python's logging framework was added in Python
2.2, and all these core parts have been there from the beginning.
However, there have historically been two problems associated with
using logging in a Django project.

Firstly, logging can seem very complicated to set up. Much of Python's
logging docs are dedicated to the various APIs that can be used to
configure the loggers, handlers, filters, and formatters that your
project will use.

This problem has been addressed by Python itself. In Python 2.7, a new
way to configure logging was introduced, called ``dictConfig``. This
is a declarative, dictionary-based format for describing logging
configuration. Since most of the logging configuration process is
really about connecting inputs to outputs and setting reporting
levels, a simple dictionary provides more than enough flexibility for
almost every logging setup.

However, having something in Python 2.7 doesn't help if you're stuck
using Python 2.4 -- so, to make sure everyone can use dictConfig,
Django has included a copy of the dictConfig library as part of
``django.utils``.

The second problem -- and the more interesting problem from Django's
perspective -- is that even if you were comfortable with Python's
logging configuration APIs, it wasn't obvious where those APIs should
be invoked from within a Django stack. Standalone programs have a
clear startup routine, which is an obvious place to put logging
configuration -- but a Django stack doesn't have an obvious 'startup'
point [1]_.

.. [1] This is a long standing feature request, and something that
   will probably be addressed in Django 1.4 as part of a refactor of
   the way applications are configured.

This second problem has been solved with a new setting -- ``LOGGING``.
This setting allows you to define a logging configuration dictionary
(in ``dictConfig`` format). When an instance of a Django project is
instantiated, this dictionary will be used to configure logging.

Logging configuration occurs right after the project settings has been
configured. This means that logging calls can be made almost anywhere
in your code, as configuration of settings is one of the first things
to occur during startup.

If you don't like using ``dictConfig`` format (or you have
particularly esoteric logging requirements), Django provides an
alternative. There is a second setting -- ``LOGGING_CONFIG`` -- that
allows you to define a callable that configures logging however you
would like. You can even use this callable to configure a `completely
different logging framework`_, or to disable the configuration of
logging altogether.

.. _completely different logging framework: http://packages.python.org/Logbook/index.html

What now?
=========

If you upgrade to Django 1.3, you don't need to make any changes to
start using logging -- it's actually used by default for all of Django's
error reporting actions that were previously hard coded. Django's 500
handler doesn't just email errors to admins anymore. Instead, it
passes the errors to a logger. The default logging configuration
handles those errors by passing them to an email handler. Want to
handle errors in some other way? Just put a configuration line in your
settings, and install a different handler.


All that is left now is to use logging in your project. `Django's
documentation on logging`_ provide lots more detail on how to use and
configure logging in your project. `Python's own logging
documentation`_ provides even more details, especially regarding the
capabilities of handlers, filters and formatters, and the various ways
that logging can be configured.

Once you've added logging to your Django projects, you can start using
other tools to analyze the data contained in your logs. Tools like
Nagios_, Arecibo_ or `Django Sentry`_ provide all manner of analysis
and alerting features that can be used to prioritize the errors and
events that your site generates.

Logs aren't just `great as a snack`_ -- they're a great way to keep on
top of what your site is doing. And Django 1.3 makes it a whole lot
simpler to use them. Enjoy your Django logging!

.. _Django's documentation on logging: http://docs.djangoproject.com/en/dev/topics/logging.html
.. _Python's own logging documentation: http://docs.python.org/library/logging.html
.. _Nagios: http://nagios.org
.. _Arecibo: http://areceiboapp.com
.. _Django Sentry: https://github.com/dcramer/django-sentry
.. _great as a snack: http://www.youtube.com/watch?v=hP0kWqJJZa4
