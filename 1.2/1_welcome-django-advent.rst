:Author:
    Jacob Kaplan-Moss


#########################
Welcome to Django-Advent!
#########################

I'm incredibly excited about the release of Django 1.2. Looking back, there've
been a few moments that turned out to be great inflection points for Django as a
project and for our community. The landing of the "magic-removal" branch and the
subsequent 0.96 release was the first of these points: we released a vastly
improved Django, and our community began to grow by leaps and bounds. And of
course the release of Django 1.0 was another such moment: the maturity of
Django, and the "1.0" label proudly proclaiming our confidence, really propelled
Django forward.

I'm fairly certain that 1.2 will be another such inflection point.

The new features -- multiple database support, model validation, vastly improved
CSRF protection, improvements to the admin UI -- are some of the most hotly
anticipated in Django's history. These features will let our users take Django
to new levels, and I can't wait to see what y'all will be able to pull off.

Over the next few weeks you'll be treated to articles discussing these new
features in depth, many written by the implementors of these features. So as to
not steal their thunder, I'll try to look at the big picture of how they all fit
together, and how they'll lay the groundwork for Django 1.3, 1.4, and beyond.

Multiple database support is shaping up to be Django 1.2's killer feature. It's
been an interesting challenge adding features of this nature: there's a natural
tension between the needs of new and small-scale developers -- who value
simplicity and ease of use above all else -- and the increasing number of
large-scale Django developers -- who need much more control, and hence more
complexity.

I think this tension between simplicity and configurability, between ease-of-use
and scale, will continue to come up. As Django matures, users will use it
to tackle tougher and tougher tasks, and we'll want Django to scale to the
challenge. At the same time, though, there'll always be many users new to
Django, and if we swamp them with complexity they'll look elsewhere, and our
community will stagnate.

In Django's new multiple database support I see that we've walked the line quite
closely. I'm proud of the job we've done balancing these competing needs:
getting started with Django 1.2 is as easy as ever, but we've vastly expanded
the "upper end" of where you'll be able to take Django. It's my hope that we'll
be able to strike similar best-of-both-worlds balances.

Beyond features, though, I'm most excited by the changes I'm seeing in our
development community. Over the Django 1.2 release cycle we've seen a lot of new
blood in our development community. Even better, as existing contributors' time
has waned, new developers have stepped up, and work's still getting done.

Much of this new blood came to use via Google's Summer of Code 2009. The year we
accepted six projects:

    * Multiple database support (Alex Gaynor).
    * HTTP and WSGI improvements (Chris Cahoon).
    * Improved localization features (Marc Garcia).
    * Model validation (Honza Král).
    * Test framework additions (Kevin Kubasic).
    * Admin UI improvements (Zain Memon).
    
The program was a smashing success: all of the projects produced good, working
code. Some has found its way into Django 1.2, and some will continue to trickle
back into trunk over the 1.3 release cycle. Many of these projects were inspired
by long-wished-for features, and the work done during the Summer of Code ended
up being what we needed to get these features over the finish line.

We've also added a couple of new committers -- Jannis Leidel and James Tauber --
and I'd wager that we'll see a few more folks get the commit bit before the
final release.

Speaking of that final release -- what of it?

The planned release date is **March 9, 2010**. We're on track for an on-time
release... but we'll never make it without your help!

Every release of Django is truly the work of thousands. We need users who can
try out our beta releases in test environments and report back to us. If you
find bugs, you can report them `on our ticket tracker`__.

__ http://code.djangoproject.com/

Even better, we'd love to have your help *fixing* bugs. Contributions on any
level -- developing code, writing documentation or simply triaging tickets and
helping to test proposed bugfixes -- are always welcome and greatly appreciated.

Django's online documentation includes pointers on `how to contribute to
Django`__

__ http://docs.djangoproject.com/en/dev/internals/contributing/#internals-contributing

We'll also be holding development sprints for Django 1.2 at PyCon US 2010. We'll
hold four days of sprints (February 22 - 25), and we'd love to have you join us
in person in Atlanta, or virtually in IRC (``#django-dev`` on
``irc.freenode.net``) or on our `mailing list`__

__ http://groups.google.com/group/django-developers

I hope you'll enjoy the rest of the series, and I hope you'll end up as excited
about the release of Django 1.2 as I am!
