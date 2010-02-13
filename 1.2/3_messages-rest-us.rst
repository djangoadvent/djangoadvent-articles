:Author:
	Jeff Croft

===========================
Messages for the rest of us
===========================

In today's world of web applications, it is frequently necessary to send
notifications to visitors. From "Your comment has been received, and is
awaiting moderation," to "Thanks for your interest, we'll send you an invite
when we launch," these little messages show up all the time, and it's nice to
have a consistent interface to sending and displaying them for your users.

Django's bundled authentication and authorization app (`django.contrib.auth`)
has always included basic functionality to send simple messages to users, but
it had some pretty serious knocks against it. Django 1.2 now includes a brand
new messaging framework, written primarily by Tobias McNulty, that makes
sending these types of notifications painless and flexible.

What was wrong with the old way?
================================

Because the legacy messaging system is part of `django.contrib.auth`, you must
install that app in order to use it. But there are plenty of instances in which
you may not need authentication and authorization, but you do need messaging.
Or perhaps you prefer to use your own auth app, instead of the one included
with Django. Or, you want to use `django.contrib.auth` but don't want to use
messages. Django 1.2's messaging framework allows for these configurations.

Additionally, the legacy system stores and retrieves these messages from the
database. While this is fine for many projects, it does mean that most every
page view incurs an additional database hit -- which is simply unnecessary, in
most cases. Django 1.2 provides options for storing messages outside the
database to avoid this performance hit.

What's more, in previous versions of Django all messages are treated equally.
There is no way to indicate a "type" or "level" for a message -- for example,
to differentiate between an error message and a success message. The new
messaging framework provides this feature.

Finally, Django 1.1's messaging system associates each message with a
particular user. Therefore, you are limited to sending messages to logged-in
users. In Django 1.2, you're able to send messages to visitors regardless of
whether or not they're signed in.

Getting started with the new messaging framework
================================================

Messaging is enabled by default for new projects created with `django-admin.py
startproject`. So if you're just starting your project with Django 1.2,
messaging is already enabled! If you've got an existing project, or create your
projects another way, enabling messaging functionality is very simple. Just
complete these three steps:

1. Add `'django.contrib.messages.middleware.MessageMiddleware'` to
   `MIDDLEWARE_CLASSES` in your settings file.
2. Add `'django.contrib.messages.context_processors.messages'` to
   `TEMPLATE_CONTEXT_PROCESSORS` in your settings file.
3. Add `'django.contrib.messages'` to `INSTALLED_APPS` in your settings file.

Sending messages
================

Sending messages to visitors with the new framework is also simple, especially
if you want to use one of Django's five built-in message levels (`DEBUG`,
`INFO`, `SUCCESS`, `WARNING`, and `ERROR`. More on message level later).

To send a success message to a visitor's request, simply call:

.. sourcecode:: python

    from django.contrib import messages
    messages.success(request, "Skadoosh! You've updated your profile!")

Likewise for the other default message levels:

.. sourcecode:: python

    messages.info(request, 'Yo! There are new comments on your photo!')
    messages.error(request, 'Doh! Something went wrong.')
    messages.debug(request, 'Bam! %s objects were modified.' % modified_count)
    messages.warning(request, 'Uh-oh. Your account expires in %s days.' % expiration_days)

Since messages are added to the request, you'll need to have access to a
request object (as you do in every Django view).

If you're upgrading code which used the legacy messaging system, there are two
steps:

1. In each view that sends messages, add `from django.contrib import messages`
   to the top of the file.
2. Replace each instance of `request.user.message_set.create(message=message)`
   with the new API, such as `message.error(request, message)`.

Displaying messages
===================

To display messages in a template, use code along these lines:

.. sourcecode:: html+django

    {% if messages %}
        <ul class="messages">
            {% for message in messages %}
                <li{% if message.tags %} class="{{ message.tags }}"{% endif %}>{{ message }}</li>
            {% endfor %}
        </ul>
    {% endif %}

It usually makes sense to put this message template code in your base template,
so all templates that extend it will display messages.

The code is nearly identical to what you may have been using with Django 1.1,
but you'll note the new `tags` attribute for each message. Django 1.2
associates each message level with a string representation for use in your HTML
template [#]_. Here, we output the tags as
classes, which can then be used for individual styling of different message
levels:

.. sourcecode:: css

    .messages li.error { background-color: red; }
    .messages li.success { background-color: green; }

If all you want to do is send some basic messages, you don't have to read any
farther. That's really all there is to it. It's simple and very flexible.

The message storage engine
==========================

Django provides a handful of backend storage systems for messages -- and it's
also easy to write your own. The `LegacyFallbackStorage` engine Django uses as
the default is sensible and appropriate for most projects. However, there are
reasons why you may want to switch engines, so it's good to understand the
options:

* `django.contrib.messages.storage.session.SessionStorage`: This engine stores
  all messages in the request's session. Therefore, it requires the
  `django.contrib.sessions` app be installed (since it's enabled by default, most
  projects are already using it). By default, sessions are stored in the
  database, so using this storage option does require a database hit each time
  you check for messages in a template (i.e. `{% if messages %}`).

* `django.contrib.messages.storage.cookie.CookieStorage`: This engine stores
  all messages in a cookie. Therefore, it does not require a hit to the database,
  providing better performance than `SessionStorage`. However, cookies can only
  be 4096 bytes in length, so when a user's message exceed 4096 bytes, messages
  will not be delivered.

* `django.contrib.messages.storage.fallback.FallbackStorage`: This engine first
  uses `CookieStorage`, but falls back to `SessionStorage` in the event all
  message could not fit in a single cookie.

* `django.contrib.messages.storage.user_messages.LegacyFallbackStorage`: This
  engine is provided for backwards compatibility with Django 1.1 and earlier. It
  works exactly like `FallbackStorage`, but also retrieves messages from the
  legacy messaging system in `django.contrib.auth`. This provides compatibility
  with any applications that haven't yet been updated to use the new messages
  framework. Like the messaging system in `django.contrib.auth`, this engine is
  deprecated, and will be removed in Django 1.4. In the meantime, it's the
  default storage method for the new messages framework. When it's removed in
  1.4, `FallbackStorage` will become the new default.

Message levels and tags
=======================

As noted earlier, Django includes five common message levels by default. Each
level is an integer. You can easily extend or modify these for your own needs.
The default message levels map to the following integers:

* DEBUG: 10
* INFO: 20
* SUCCESS: 25
* WARNING: 30
* ERROR: 40

To add your own message level, define a constant and then use the
`add_message()` method to send a message using it:

.. sourcecode:: python

    CRITICAL = 50
    messages.add_message(request, CRITICAL, 'OH NOES! A critical error occurred.')

If you want to use the message level in HTML or CSS, as we did above, you'll
also want to add `MESSAGE_TAGS` to your settings file to provide a mapping
between your levels and the tag you want outputted in your templates:

.. sourcecode:: python

    MESSAGE_TAGS: { 50: 'critical' }

In conclusion
=============

The new messaging framework included in Django 1.2 is not a particularly
complex piece, but it does provide functionality that is absolutely essential
in today's word of web apps, and it does so in an elegant, clean, and simple
way. What's more, it's completely backwards-compatible with the legacy system,
so you don't have to worry about old code or third-party apps breaking when you
upgrade to 1.2. Be aware, though, that this compatibility layer will be removed
in 1.4 -- slated for release sometime in 2027. [#]_

Joking. Enjoy Django 1.2, everyone. It really is a great update to our favorite framework.

.. [#] Ostensibly for CSS and JavaScript hooks.
.. [#] *Late* 2027, if I had to guess.
