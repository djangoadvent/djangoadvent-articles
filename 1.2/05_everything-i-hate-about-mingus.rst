:Author:
	Kevin Fricovsky

##############################
Everything I hate about Mingus
##############################

Overview
========

Mingus is a small Django project, created as an experiment in practicing one of
the key features of Django -- reusable apps. Mingus defines no models itself,
but currently leverages 30+ reusable apps to provide one complete blog engine
project. 

This article is about the obstacles faced and lessons learned managing an 
application that relies on so many reusable apps, and experiences in 
managing a small, open source project. [#]_ It has nothing to do specifically 
with all the awesome that exists in Django 1.2.

What is Mingus?
===============

Mingus is a blog engine. The act of developing yet another blog engine from 
the ground up is a chore really. Mingus exists to do that chore for you. How 
nice of Mingus, no?

So, why is the project a (very) small success? Four key reasons:

#. The concept is simple -- a blog engine
#. The concept is intriguing -- use only reusable apps to provide all of its
   features (blogging, commenting via Disqus, admin image cropping, inlines,
   WYSIWYG editor, debugging, contact form, etc.)
#. It's a learning tool on how developers can leverage reusable apps in their
   project, and a forced introduction to virtualenv and pip.
#. It has a minimal, attractive template system.

There is one thing that Mingus really strives to do, and that is to take all of 
the combined complexity of 30+ reusable apps, and reduce it into one project 
that's brain dead simple to get up and running. Literally, this is what it 
takes to get started:

.. sourcecode:: sh

    mkvirtualenv myblog —-no-site-packages
    workon myblog
    cdvirtualenv
    git clone git://github.com/montylounge/django-mingus.git
    cd django-mingus/mingus
    pip install -r stable-requirements.txt
    cp local_settings.py.template local_settings.py
    ./manage.py syncdb
    ./manage.py loaddata fixtures/test_data.json
    ./manage.py runserver

For those not familiar, the above commands `mkvirtualenv`, `workon`, and
`cdvirtualenv` requires you have `virtualenvwrapper`_ installed. I included 
the reference to `mkvirtualenv` and the others as a dependency in the 
documentation as a way to introduce developers to the `virtualenvwrapper` 
project if they were not already familiar. [#]_ I believe it's a terrific tool 
that every Python developer should at least be familiar with, and I snuck in 
the reference so that maybe it would introduce a new user to the project. 
You'll see that tactic used quite often if you take a look throughout the code 
base.

Now we know it's really easy to get started with Mingus, so let's dive in...


Be "that guy" (or girl)
=======================

If you ever started an open source project that gained even a tiny bit of 
momentum, then you've received feature requests of varying kinds. 
Eventually someone will come along who is using your project but they 
need a few extra features that would help him/her along. 
Their requests end up in your email inbox and them on your issue 
list quite often.

As a project maintainer, this is where you need to start making some tough
decisions. Are the features requested within the scope of your project's
mission? Do they help make your project more complete? Or are these requests
solely for the benefit of this one individual? Learning when to say "no" to a
request is essential to the successful management of any project. But if the
requests are practical, and if you care, then you are sensitive to things that
should be done correctly and now you've added another action item to add 
to your to-do list.

So you're a good project lead and you fix the bug or add that feature. And if
you really care, while you are refactoring code and you're using your project
daily [#]_, then you start thinking like "that guy" and you start seeing ways
to make your project better. So you spend more time adding new features that
you know are reasonable, and relative to your project's focus. Features that
you may have first overlooked, or simply shrugged off.

I hate that guy because he adds to my to-do list, but I love that guy because
he's making me a better developer, and he's making my project better. He's my
QA. He's gathering my requirements. Mingus wouldn't be the tiny success that it
is if it wasn't for that guy(s).

My advice to you — be that guy (or girl) and make me hate you for it.


Don't make me restart Apache
============================

If there was one feature that I believe the Django admin is missing I would 
say application wide settings management.

The one request I receive on almost every single project I have every been on
is the ability to manage application settings via the admin. I don't care if
it's a Django project or a ASP.Net project -- every single admin user has
needed the ability to manage application wide settings via the admin UI. But
there's no current best practice in managing application wide settings in
Django. We need one, and it needs to be available via the admin.

Well, actually, there is a best practice out there, but it ends up being a pain
point. The current best practice is to place application settings in the
`settings.py` file of your project. Here's an example of how to best implement
retrieving that value:

.. sourcecode:: python

    from django.conf import settings
    post_list_count = getattr(settings, "post_list_count", 20)

The above uses `getattr` to retrieve the `post_list_count` value from the
settings instance. If one doesn't exist it defaults to 20. This is terrific and
follows the best practice of "sane defaults".

Now, take this example, and lets assume every app in your project (Mingus for
example) requires that one setting is defined per app in your settings.py -- in
Mingus' case you would now have 30 additional values to manage in
`settings.py`. It's not horrible, but it could lead to an extremely complex
`settings.py` file when we're all looking to minimize our settings files,
aren't we? 

When following the above convention, if a settings value change is requested, 
a developer/sysadmin then needs access to the server to update the 
`settings.py` file themselves. Moreover, whoever handles the change request 
also needs to restart the web application server for the change to take affect. 
This is less than ideal.

Lucky for us, two reusable apps attempt to provide that solution: 

#. Django-DBSettings_
#. Django-LiveSettings_

I'm not sure which, but one of these solutions (or some of their shared
concepts) should become the convention and ideally an approved contrib
application. 

Sure there are simple settings value:

.. sourcecode:: python

    BLOG_PAGE_SIZE = 20

And there are more advanced:

.. sourcecode:: python

    DEBUG_TOOLBAR_PANELS = (
        'debug_toolbar.panels.version.VersionDebugPanel',
        'debug_toolbar.panels.timer.TimerDebugPanel',
        'debug_toolbar.panels.settings_vars.SettingsVarsDebugPanel',
        'debug_toolbar.panels.headers.HeaderDebugPanel',
        'debug_toolbar.panels.request_vars.RequestVarsDebugPanel',
        'debug_toolbar.panels.template.TemplateDebugPanel',
        'debug_toolbar.panels.sql.SQLDebugPanel',
        'debug_toolbar.panels.signals.SignalDebugPanel',
        'debug_toolbar.panels.logger.LoggingPanel',
    )

The solution? Beyond a simple key/value store each application could provide a
handler. Django would provide default handlers of course...
`IntegerSettingHandler`, `StringSettingHandler`, etc. But an application like
`django-debug-toolbar`_ would provide a `DebugToolBarHandler`. This will allow 
the Settings app to have a standard API that any application can interface with 
via the Settings API but each custom application provides it's own custom 
handler logic to execute its rules on its own. And maybe I'm just crazy?

There's also the extra query factor for retrieving these values if they exist
in the backend store. So a sensitivity towards performance is required.

Right now if I want to add an application to Mingus I try to think about the
needs of the non-technical end user. Would they want to be able to change this
setting via the admin? If yes, and the reusable app defines a `settings.py`
required value, I have a tough decision to make. Do I fork that app and add the
setting to a model which can be updated in the admin [#]_ or do I just fold and
give in, including the app and dropping another value into the project's
`settings.py`?

Having a contrib app that resolves these issues would reduce complexity,
maintenance, needless forks of code bases, and improve app flexibility and
integration.

You can get with this Setting or that Setting
=============================================

As if the previous global settings management discussion wasn't exciting
enough, it's now time to talk about managing those settings files. 
 
In Mingus I package two settings files:

#. settings.py
#. local_settings.py

The former maintains all the application wide settings. The latter is the
override, allowing the developer to override various settings on her machine,
and allowing the various stages of your environment (dev, staging, production)
to have their own `local_settings.py` file defining environment specific
setting values (think database settings, filesystem settings, debugging, etc). 

This has been a convention I've come across a few times before when looking at
other projects, so I stuck with this basic pattern.

But it's not the final answer. If you were to take a look at Daniel Lindsey's
blog post `Better Local Settings`_ you'll see one proposed solution. Then read
the comments of his post and you'll see a few other solutions, highlighting the
fact that we need a standard. In fact, the popular DjangoDose_ podcast proposed 
their solution in their `Handling Development, Staging, and Production 
Environments`_ using the FLAVOR concept. But again, take a peak at the comments
and you'll see another handful of alternative solutions used by other 
developers.

A contributer to Mingus suggested I take a look at the Transifix_ team's 
documentation `Using a list of conf files`_ on how they manage their settings 
files as a best practice, which looks interesting as well. The simple fact 
that wiki page for various solutions in managing your settings files even 
exists highlights the need for a standard.

One project that recently found its way on my radar is `Django-Config`_ from 
Nowell Strite, Shawn Rider and now supported by Tareque Hossain. Strite and 
Rider both work at PBS and recently detailed the obstacles they run into 
supporting the various projects and reusable apps across their infrastructure 
with their `Pluggable, Reusable Django Apps: A Use Case and Proposed Solution`_ 
presentation at DjangoCon 2009.

Django-Config defines itself as "...an easy way to maintain multiple 
configurations for django. It relies on the concept of having a shared 
configuration file (base) and a per user/ server custom configuration file 
(dev1/ dev2/ local/ staging). settings.py combines the base & custom 
configuration and loads it up." I have yet to give the project a run myself
but assuming the complexity of the infrastructure that PBS maintains 
I'm going to believe that there's a few nuggets of tested and refined goodies
in there.

So I'm left not knowing what to do. For now, I'll keep with the basic
implementation.


Static Media? No you didn't!
============================

Anyone who has ever authored a Django reusable app has asked themselves the
question, where do I put the static media? What do I name the directory? Do I
name it /media/ or do I name it /static/? Where do I place it on my file
system?

A perfect example is Simon Willison's django-cropper reusable app I recently
integrated into Mingus. Willison recently left this git `commit message`_,
"Finally managed to get the package to include the template... no idea what I
should do with the static file dependencies though". It's a good question.
What does a developer do with static media dependencies? Do they include the 
files in their project? Do they tell the user to go download them from XYZ? If 
they do include the files, where do they place them in their app?

I believe there is an answer... Django-StaticFiles_.

The project stems from the Pinax_ project that faces this same obstacle in a
much larger scale. So if anyone knows a solution, the Pinax crew would. I'm not
going to dive into the finer details of the project, as it provides a terrific
set of features and functionality, but what it outlines in its implementation
is an easy to follow standard for reusable apps and static media management. 
Maybe it should become a contrib app, or at least the convention we all look 
to?

Upload this pal
===============

While we're here talking about media management, let's also talk about files
uploaded via the Django admin. I believe we should also have a default
convention here too -- the /uploads/ directory off MEDIA_ROOT. Far too often
I'll grab an application that has this:

.. sourcecode:: python

    photo = models.ImageField(upload_to="/images/")

That helps no one. The convention should be `MEDIA_ROOT + "/uploads/app_name/"`
as the default, and any directory defined in the `upload_to` parameter is
appended to the default, like so (in my photobooth app):

.. sourcecode:: python

    photo = models.ImageField(upload_to="images")

By default this would result in `MEDIA_ROOT + /uploads/photobooth/images/` file
path.

I'm simplifying the underlying complexity, obviously, but I do believe a sane
default would provide better asset management, and again make reusable app
integration less invasive for these cases.


i18n gets no respect
====================

I'll make this as short and sweet as possible. The internationalization_
features in Django are amazing. Having built one multi-lingual site from the
ground up, and benefitting from the features Django provides out of the box 
for this i18n, it's a damn shame more reusable apps don't internationalize
their app from the start (and I'm to blame here myself -- let's just be 
honest).

Here's the two simplest ways to at least lay the groundwork for 
internationalizing your application. Let's take a `models.py` for this example.
All you need to do is this:

.. sourcecode:: python

    from django.utils.translation import ugettext_lazy as _
    ...

    class Post(models.Model)
        description = models.TextField(_('description), help_text=_('The description of your post'))

        class Meta:
            verbose_name = _('post')
            verbose_name_plural = _('posts')
    ...

Now in your templates all you need to do for the text laying around is this, 
example landing.html:

.. sourcecode:: html+django

    {% load trans %}

    <h4>{% trans "Blog roll" %}</h4>

And that's about it. As always, there's a little more under the hood, so make
sure to read the docs which covers everything you need to know in getting
started, but for the most part the above gets your app 80-90% of the way there. 

And if you are deploying a multi-lingual application you will want to take a
look at these apps:

* Django-Rosetta_
* Django-DataTrans_

And take a look at this excellant article which reviews a handful of reusable
apps to help you with i18n integration -- `Dynamic Translation Apps for
Django`_. The fact is that if you are not internationalizing your app then you
are a bad person. No, but seriously, if you aren't internationalizing your app
you are creating a headache for another developer, and more importantly you are
also limiting the potential adoption of your project. So be a good person and
internationalize that bad boy.


Ain't nothing but a Migrations thing
====================================

We have to start including South migrations in all our reusable apps we
publish. Or we need my pony request to be fulfilled (discussed below). I've
pitched this pony request once before in my `South and Reusable Apps`_ post but
I wanted to reiterate the importance of the community selecting a migration
tool. 

As we discussed on the `Reusable Apps in Django Panel`_ on DjangoDose the
current best-in-show migration tool is South_. There is currently no easy way
to migrate a collection of reusable apps since migration management isn't a
discipline I've found practiced in most apps. And again, I'm to blame for this
as well. But no more. Moving forward I'm putting my eggs in the South basket. 

Now, South could provide a feature that makes this rather easy for us
developers. That is the proposition I made in the aforementioned blog post.
Andrew Godwin, the author of South, commented that the solution for migrating
reusable apps that don't employ South themselves is already in the works in a
forthcoming version of South. So all hope is not lost. This feature would allow
us developers to generate South migrations for the reusable apps we leverage
even if they don't make use of South themselves. Terrific!

Right now Mingus provides only one migration (raw sql) and that's because it
wasn't until recently that people started using Mingus as their blog engine.
And knowing this it would be negligent of me to not provide migrations for
these users looking to upgrade to the next Mingus release. So at least they
have raw sql to work with, but it's not the right answer. The right answer is a
standard migration tool we all use.


Cache Keys Rule Everything Around Me
====================================

The Django `cache framework`_ provides a tremendous amount of caching
functionality and flexibility. The one thing I often hear developers
reiterating is to make sure your cache keys are named properly so to avoid
cache key conflicts. You want unique cache keys that can be recalled easily for
cache validation/invalidation, querying, etc.

So why not include a helper to ease this? That exactly what I did when I added
`create_cache_key`_ to django-sugar_. The method actually combines the code of
one blog post and a reusable app:

* `Improving Django Cache`_
* `Django-Caching`_

Here's a look at the api:

.. sourcecode:: python

    from blog.models import Post
    slug_val = 'some-slug'
    mykey = create_cache_key(Post, 'slug', slug_val)
    obj = cache.get(mykey)

What the above `create_cache_key` does is accept either a Model or Manager as
its first argument, the field you are interested in as its 2nd argument, and
the field value as its 3rd argument. Based on that it can generate, and
regenerate a cache key. The benefit here is that it isolates the logic for
remembering cache key names. It handles the construction for you.

This may not be the best solution possible or the most complete solution, but
it's a solution that begs the question: why don't we have a similar utility
method in Django itself that we reference by default so we don't have to be
concerned about clashing cache key values in our apps? 

Conclusion
==========

The reason I love hacking on Mingus is Django... I love Django, and Python.
So the above "hates" are really just small bumps in the road of an amazingly
smooth ride that Django provides.

The future of Mingus is a final 1.0 release which will include any bug fixes 
that pop up, more documentation, more tests, and other than that I don't
think there's much left to add to something that's not really anything more
than a concept project. I believe the current feature set is final.

For those who manage any open source project, big or small, I tip my hat to you
and thank you. Just like you I'm excited for all the amazing things coming in 
Django 1.2 and as a consumer of such a terrific open source project, I feel lucky 
that I get to work with Django daily. In ending this I just realized I was 
"that guy" for most of this article and I hate myself for it.


.. _`virtualenvwrapper`: http://www.doughellmann.com/projects/virtualenvwrapper/
.. _`101 Things I Learned in Architecture School`: http://www.amazon.com/101-Things-Learned-Architecture-School/dp/0262062666
.. _`Django-DBSettings`: http://github.com/sciyoshi/django-dbsettings
.. _`Django-LiveSettings`: http://bitbucket.org/bkroeze/django-livesettings/
.. _`Better Local Settings`: http://toastdriven.com/fresh/better-local-settings/
.. _`Handling Development, Staging, and Production Environments`: http://djangodose.com/articles/2009/09/handling-development-staging-and-production-enviro/
.. _`Pluggable, Reusable Django Apps: A Use Case and Proposed Solution`: http://blip.tv/file/3040424
.. _`Django-Config`: http://github.com/tarequeh/django-config
.. _`DjangoDose`: http://djangodose.com
.. _`Transifix`: http://trac.transifex.org/
.. _`Using a list of conf files`: http://code.djangoproject.com/wiki/SplitSettings#UsingalistofconffilesTransifex
.. _`DjangoProject.com`: http://djangoproject.com
.. _`Internationalization`: http://docs.djangoproject.com/en/dev/topics/i18n/
.. _`Django-Rosetta`: http://code.google.com/p/django-rosetta/
.. _`Django-DataTrans`: http://github.com/citylive/django-datatrans
.. _`commit message`: http://github.com/simonw/django_cropper/commit/ef9e5334a333f40668dccfb9d6d00ef9ce72e0a2
.. _`Django-StaticFiles`: http://github.com/jezdez/django-staticfiles
.. _`Pinax`: http://pinaxproject.com
.. _`Dynamic Translation Apps for Django`: http://www.muhuk.com/2010/01/dynamic-translation-apps-for-django/
.. _`South and Reusable Apps`: http://blog.montylounge.com/2009/oct/21/south-and-reusable-apps/
.. _`Reusable Apps in Django Panel`: http://djangodose.com/blog/2009/10/reusable-application-panel/
.. _`South`: http://south.aeracode.org/
.. _`Cache Framework`: http://docs.djangoproject.com/en/dev/topics/cache/
.. _`create_cache_key`: http://github.com/montylounge/django-sugar/blob/master/sugar/cache/utils.py#L27
.. _`Django-Sugar`: http://github.com/montylounge/django-sugar/
.. _`Improving Django Cache`: http://richwklein.com/2009/08/04/improving-django-cache-part-ii/
.. _`Django-Caching`: http://github.com/mmalone/django-caching/
.. _`django-debug-toolbar`: http://robhudson.github.com/django-debug-toolbar/

.. [#] Even if it's an itsy bitsy one.
.. [#] Since my docs reference these, I assume that by following the docs you
   forced yourself to play with these excellent tools.
.. [#] Eating your own dog food.
.. [#] At this point it's arguably no longer reusable, or actually maybe more
   reusable now that I think about it.

