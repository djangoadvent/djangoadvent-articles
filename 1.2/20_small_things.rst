:Author:
	Jacob Kaplan-Moss

############
Small Things
############

Size doesn't matter
===================

Throughout the advent, we've been covering some of the big new__ features__ in
Django 1.2. Those are all awesome, of course, but some of my favorites parts of
Django 1.2 aren't big. There's a bunch of really sweet little improvements that
taken together really add up.

Here are some of my favorites:

__ http://djangoadvent.com/1.2/multiple-database-support/
__ http://djangoadvent.com/1.2/messages-rest-us/

Email backends
--------------

Django 1.2 gives you complete control over `how Django sends email`__.

__ http://docs.djangoproject.com/en/dev/topics/email/#topic-email-backends

Django ships with a handful of standard backends, including the default SMTP
backend and backends that log emails to files, the console, or memory.

We added this feature primarily to support those running Django on sandboxed
hosting environments like Google's `App Engine`__, but I think it'll prove
useful in a bunch of other situations.

__ http://code.google.com/appengine/

The API's very simple: an email backend is a class that implements a handful of
methods:

    * ``send_messages(email_messages)``, which sends a list of EmailMessage__
      objects.

    * ``open()`` (optional), which starts an e-mail-sending connection.

    * ``close()`` (optional), which closes the current connection.
    
__ http://docs.djangoproject.com/en/dev/topics/email/#django.core.mail.EmailMessage

For example, you could easily redirect email to a RESTful API with something
like the following (which uses the httplib2__ library):

.. sourcecode:: python

    import httplib2
    from django.conf import settings
    from django.utils import simplejson

    class HttpPostEmailBackend(object):
        def send_messages(self, email_messages):
            http = httplib2.Http()
            
            # Convert the list of messages to JSON to posting
            body = simplejson.dumps([m.__dict__ for m in messages])
            headers = {'content-type': 'application/json'}
            
            resp, content = h.request(settings.EMAIL_TARGET_URL, "POST", body=body, headers=headers)
            
            # In Real Life we'd have error handling here -- the request could
            # fail or time out. That's left as an exercise for the reader.
            
            # Return the number of successfully sent messages.
            return len(messages)

__ http://code.google.com/p/httplib2/

Then just wire it up in your settings file:

.. sourcecode:: python

    EMAIL_BACKEND = 'path.to.HttpPostEmailBackend'
    EMAIL_TARGET_URL = 'http://example.com/email'

Read-only admin fields
----------------------

You can now make fields read-only in the admin with the ``readonly_fields``
option:

.. sourcecode:: python

    class PersonAdmin(admin.ModelAdmin):
        readonly_fields = ['age']

These fields will show up in the admin on the add/change pages, but won't be
editable. I see this being most useful when used with fields containing
calculated data, or other such de-normalized information.

Smarter if tag
--------------

Designers and template developers, this one's for you:

One of Django's core design philosophies is that `template languages shouldn't
try to be programming languages`__. To that end, we've kept the template
language deliberately simplisitic so that you'd be more or less forced to push
advanced logic up into the view.

__ http://docs.djangoproject.com/en/1.1/misc/design-philosophies/#don-t-invent-a-programming-language

It's a delicate line, though, and over the years it's become clear that the
``if`` tag took "simplicity" a bit too far. Thus, Django 1.2's ``if`` tag
has been made much more powerful. It now supports comparison operators, meaning
you can do things like:

.. sourcecode:: html+django

    {% if user.username == "jacob" %}
      ...
    {% endif %}
    
    {% if driver.bac >= 0.8 %}
      ...
    {% endif %}
    
    {% if athlete in team.roster %}
      ...
    {% endif %}

The tag supports ``==``, ``!=``, ``<``, ``>``, ``<=``, ``>=`` and ``in``, all of
which work like their Python equivalents. You can chain logic together with
``and``, ``or`` and ``not``.

This means that ``ifequal`` and ``ifnotequal`` are no longer necessary, though
they're still available for backwards compatibility.

CSS classes on required/erroneous form rows
-------------------------------------------

It's pretty common to style form rows and fields that are required or have
errors. For example, you might want to present required form rows in bold and
highlight errors in red.

Django 1.2 now makes it easy to add ``class`` attributes to the generated HTML
for form fields and rows; you can then use CSS to style them accordingly. You'll
specify these class names in your form definitions:

.. sourcecode:: python

    class ContactForm(Form):
        error_css_class = 'error'
        required_css_class = 'required'
        
        # ... and the rest of your fields here.

This'll make the generated HTML contain ``error`` or ``required`` classes as
necessary. That is, if you use ``{{ form.as_table }}``, the generated HTML might
look something like:

.. sourcecode:: html

    <tr class="required"><th><label for="id_subject">Subject:</label> ... </th></tr>
    <tr class="required"><th><label for="id_message">Message:</label> ... </th></tr>
    <tr class="required error"><th><label for="id_sender">Sender:</label> ... </th></tr>
    <tr><th><label for="id_cc_myself">Cc myself:</label> ... </th></tr>
    
And you can then target those rows in CSS:

.. sourcecode:: css

    tr.required { font-weight: bold; }
    tr.error { background-color: red; }

Cached template loading
-----------------------

Quick, pop quiz:

Given these settings:

.. sourcecode:: python

    INSTALLED_APPS = ['app1', 'app2']
    TEMPLATE_DIRS = ['/dir1/', '/dir2/']
    
and this line of code:

.. sourcecode:: python

    return render_to_response('example.html')
    
and an ``example.html`` containing:

.. sourcecode:: html+django

    {% extends base.html %}
    
    {% block content %} Hi! {% endblock %}
    
what's the worst-case number of many disk operations Django must perform to
render the response?

...

The answer's *eight*. Why?

Well, Django has to find ``example.html``. To do so, it'll consult the `template
loaders`__. By default, these will first look in each application's directory
for the template, and then in each of the ``TEMPLATE_DIRS``. So, to find
``example.html``, Django will try to load:

    * ``<app1's path>/templates/example.html``
    * ``<app2's path>/templates/example.html``
    * ``/dir1/example.html``
    * ``/dir2/example.html``

__ http://docs.djangoproject.com/en/dev/ref/templates/api/#loader-types

Next, because ``example.html`` uses inheritance, Django must repeat the exercise
for ``base.html``.

These disk accesses are fairly cheap, but *not* free. As sites group, you'll
have many more applications, more template directories, and probably more
complex template inheritance. It's not unusual to see a single template load
operation force over 100 disk I/O requests.

Yuck.

To help mitigate this problem, Django 1.2 now ships with a cached template
loader. This wraps existing template loaders, so unknown templates are looked up
normally. However, once the template's been found, the cached loader stores the
compiled template in memory, returning it directly for subsequent requests for
the same template. That is, each subsequent request doesn't even need to touch
the disk.

To enable the cached template loader, you'll put something like this in your
settings:

.. sourcecode:: python

    TEMPLATE_LOADERS = (
        ('django.template.loaders.cached.Loader', (
            'django.template.loaders.filesystem.Loader',
            'django.template.loaders.app_directories.Loader',
        )),
    )
    
If you've got other custom template loaders you can add 'em to the inner list
and they'll be wrapped by the cached loader as well.

.. warning::

    All of the built-in Django template tags are safe to use with the cached
    loader, but if you're using custom template tags that come from third party
    packages, or that you wrote yourself, you'll need to `ensure that the
    implementation for each tag is thread-safe`__.
    
__ http://docs.djangoproject.com/en/dev/howto/custom-template-tags/#template-tag-thread-safety

Et cetera
---------

These are just my favorite small changes coming in Django 1.2. There are many
other small improvements, including:

    * A new `big integer field`__.
    
    * `Customizable syntax highlighting`__ for ``manage.py`` output,
      including highlighting of the development server output.
    
    * A new ``--failfast`` flag to ``manage.py test`` that'll stop the running
      tests and report immediately after the first test failure.

    * And many more.
    
__ http://docs.djangoproject.com/en/dev/ref/models/fields/#django.db.models.BigIntegerField
__ http://docs.djangoproject.com/en/dev/ref/django-admin/#syntax-coloring
