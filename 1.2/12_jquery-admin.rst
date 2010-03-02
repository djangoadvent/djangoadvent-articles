:Author:
	Zain Memon

###################
jQuery in the Admin
###################

`Changeset r12297`_ dropped a bomb on everyone: a new file in the
``django/contrib/admin/media/js`` directory called ``jquery.js``. This change
ushers in a new era for the Django admin; an era of fancy new features, pretty
widgets, and better usability.

The case for jQuery
===================

Despite being discussed in the past, integration of jQuery into the admin
wasn't on the map until the GSoC `admin-ui proposal`_ this summer, which
suggested some UI-heavy features that would be difficult without the use of a
framework. 

It’s easy to see the need for a Javascript framework in the admin. Raw
Javascript in any decently sized project slowly approaches the size and
complexity of a framework. That process becomes a minefield of dealing with
cross-browser issues, workarounds, and normalization; a framework
short-circuits that pain and lets developers focus on building cool stuff. We
live in the future, and there’s just no good reason to write Javascript
straight to the DOM APIs when we could be adding useful functionality (with
fewer bugs) instead.

It’s also easy to see why jQuery is the framework of choice: jQuery has
`soundly won`_ the JS framework wars. Almost 30% of the sites tracked by
`builtwith.com`_ use jQuery. Picking something else would mean fewer
knowledgeable devs, less available code, and a smaller community of support.

This doesn’t mean Django endorses jQuery
========================================

There's some concern in the community that the inclusion of jQuery appears to
be an endorsement of one framework over another -- that's not the case at all.

Django proper remains blissfully agnostic to toolkit choice. The admin is an
optional app designed for end-users, not other developers. Data entry clerks
are unaffected by the admin’s tooling, and developers are not constrained by
it.

If you're modifying the Django admin, jQuery stays out of your way. For
example, the admin uses ``jQuery.noConflict()`` to make sure the ``$`` variable
isn’t polluting your namespace, so you can use Prototype’s ``$()`` in your own
widgets without any conflicts. Also, jQuery is only loaded on pages with
widgets that require it.

Your apps can still use whatever makes sense for you. For the Django admin,
including a well-tested, proven framework makes more sense than to add even
more undocumented, unvetted spaghetti Javascript.

This does mean Django’s admin will grow much faster
===================================================

The front-end of the admin has stagnated a bit. A perfect storm is brewing for
some kick-ass new front-end features: jQuery, a newly (re)appointed `design
czar`_, and a general focus in the community on better usability.

Django 1.2 mainly focused on laying the groundwork for future admin-ui
improvements and most visible changes won't be felt until Django 1.3 and later.
However, a few things did make it in.

New features
============

One of the first jQuery features in the admin is the dynamic addition of new
inlines. For a demo of this functionality, check out this screencast:

.. raw:: html

	<video width="624" height="464" controls="controls" autobuffer="autobuffer">
		<source src="http://djangoadvent.s3.amazonaws.com/advent/002/inlines_624x464.mp4" type='video/mp4; codecs="avc1.42E01E"'>
		<source src="http://djangoadvent.s3.amazonaws.com/advent/002/inlines_624x464.ogv" type='video/ogg; codecs="theora"'>
		<object width="624" height="488" type="application/x-shockwave-flash" data="http://djangoadvent.s3.amazonaws.com/advent/002/player.swf?image=http://djangoadvent.s3.amazonaws.com/advent/002/inlines_poster.png&amp;file=http://djangoadvent.s3.amazonaws.com/advent/002/inlines_624x464.mp4">
			<param name="movie" value="http://djangoadvent.s3.amazonaws.com/advent/002/player.swf?image=http://djangoadvent.s3.amazonaws.com/advent/002/inlines_poster.png&amp;file=http://djangoadvent.s3.amazonaws.com/advent/002/inlines_624x464.mp4" />
			<img src="http://djangoadvent.s3.amazonaws.com/advent/002/inlines_poster.png" width="624" height="464" alt="Dynamic Admin Inlines"
			     title="No video playback capabilities, please download the video below" />
		</object>
	</video>
	
	<aside>Download Video: <a href="http://djangoadvent.s3.amazonaws.com/advent/002/inlines_624x464.mp4">MP4</a> | <a href="http://djangoadvent.s3.amazonaws.com/advent/002/inlines_624x464.ogv">Theora (Ogg)</a></aside>


A lot of implemented features missed the cut for Django 1.2, like drag-and-drop
reordering of inlines for models with an ordering field, and an autocomplete
widget for Foreign Key and M2M relations to be used instead of the select
drop-down (or raw_id_fields). Those features (and others) will most likely land
in Django 1.3. 

These are exciting times for the users of the Django admin. There’s already
some admin hotness in 1.2, and it’s only going to get better by the time 1.3
ships. jQuery it up!

.. _Changeset r12297: http://code.djangoproject.com/changeset/12297
.. _soundly won: http://www.google.com/trends?q=jquery,+mootools,+dojo,+prototype+js&ctab=0&geo=all&date=all&sort=0
.. _builtwith.com: http://trends.builtwith.com/javascript/JQuery
.. _`design czar`: http://groups.google.com/group/django-developers/browse_thread/thread/18bca037f10769e9/50fc65fe85746197
.. _`admin-ui proposal`: http://groups.google.com/group/django-developers/browse_thread/thread/1edf77c9c8b1101d/4ecda5e4c982c7e1

