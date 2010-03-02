:Author:
	Andrew Godwin

###################
Django 1.2 and CSRF
###################

CSRF, or Cross-Site Request Forgery, is perhaps one of the most overlooked
classes of security vulnerabilities around. SQL injection attacks and
cross-site scripting attacks are generally well-understood by any web developer
worth their salt, but CSRF attacks often catch them unawares.

For those unfamiliar with web security in general, CSRF attacks essentially
work by using the user's browser, and their pre-existing sessions and cookies,
to call other sites with malicious data [#]_. They're not a server exploit,
*per se*; they can only affect data on the target site that the user has access
to.

Don't let that fool you, though; there's a large range of nasty things you can
do with CSRF, from changing a user's profile to include spam links, to using an
administrator's account to delete pages, or slip in backdoors. There have even
been remote code execution exploits; it essentially allows the attacker access
to anything you have behind authentication or permissions.

Thankfully, due to the prescience of the Django developers, Django has shipped
with a ``CSRFMiddleware`` for quite a while now, albeit a slightly broken and
ill-designed one. To understand more, you first need to understand the basic
vector of a CSRF attack.

Anatomy of a CSRF Attack
========================

CSRF works by making the user's browser submit a request (either ``GET`` or
``POST``, depending on the target) to the site being attacked, with the content
of the request being chosen by the attacker.

For example, imagine we have an online bank, with a form that allows us to
transfer our money. I want to send some money to Alice (account number
00000001), so I fill out the form, and my browser submits this: [#]_

::

 POST /banking/transfer/ HTTP/1.1
 Host: www.andrewsbank.com
 
 amount=100&recipient=00000001
 
Now, imagine I don't log out of my banking session [#]_, and browse to a
malicious site. The site presents me with a button marked "Continue"; however,
this is the HTML

.. sourcecode:: html

 <form action="http://www.andrewsbank.com/banking/transfer/" method="POST">
     <input type="hidden" name="amount" value="100" />
     <input type="hidden" name="recipient" value="00000002" />
     <input type="submit" value="Continue" />
 </form>
 
As you can see, while the button might look innocuous, it's actually sending
money to Bob's account (00000002).

This isn't a particularly sophisticated attack, nor is it a very sophisticated
bank (I'd recommend Alice move her money elsewhere). Still, it should hopefully
get you thinking about these kinds of attacks, and what they're capable of
[#]_.

There's plenty of resources about CSRF attacks on the web if you want to know
more; the `Wikipedia article
<http://en.wikipedia.org/wiki/Cross-site_request_forgery>`_ gives a good
grounding, and there's a `good article
<http://www.codinghorror.com/blog/2008/10/preventing-csrf-and-xsrf-attacks.html>`_
by Jeff Atwood.

Protecting against CSRF
=======================

So, how does one stop CSRF attacks? The underlying method is to make sure every
request originated from a real use of your site, and isn't spoofed.

The general method is to generate a token for every form (a *nonce* in security
parlance), tied to the user's session, and to check the user's ``REFERER``
header. Tying it to the session is crucial; without it, the attacking site
could simply perform a request to your site itself and obtain a valid token.

However, if you do check the session, and make sure you issued the token
relatively recently, you can be reasonably sure it's a valid request. There's
never really any certainty in computer security, and new attacks are being
discovered all the time [#]_, but you're still a lot better off.

Django and CSRF
===============

Now, back to the aforementioned ``CSRFMiddleware``. The version of this shipped
with Django 1.1 was lacking, at best; it took the output of your site, ran a
regular expression over it that looked for ``<form>`` elements, and then added
an ``<input>`` element containing the CSRF token, which incoming ``POST``
requests were checked for.

As well as the obvious issue of it just running regular expressions over and
inserting content into your response stream, it also added tokens to every
POSTable form on the page - including ones which submitted to a third-party
site, sending a valid CSRF token along with the other data. Armed with that
token, the third-party site could then have used it to mount a CSRF attack on
that user.

There were also other problems with the old CSRF protection; for example, it
required ``SessionMiddleware`` to be enabled, and it wasn't really possible to
enable it only for one app, as middleware is global.

What's Changed?
===============

In Django 1.2, the regex-happy ``CsrfMiddleware`` has gone, and a new one has
appeared in its place, sporting more features along with a distinct lack of
regular expressions.

The new CSRF protection also requires that you insert the ``{% csrf_token %}``
tag inside any ``<form>`` elements you want to have CSRF protection, rather
than just adding it to all forms. While this means you have to remember to add
the tag to all forms which submit to CSRF-protected views, it also means that
you won't accidentally send valid tokens off to third-party sites, like before.

In addition, the new protection also doesn't need to be enabled globally to be
effective; there's a new ``django.views.decorators.csrf.csrf_protect``
decorator that you can use to enable CSRF on a per-view basis, and which all of
the core Django apps now use, meaning your admin pages are secure even if
you've turned off the middleware.

CSRF protection has also been promoted from being a contrib app to being part
of Django core; this is because we believe that, much like auto-escaping, CSRF
is a vital part of what a web framework should provide; it's very easy to get
wrong if you implement it yourself.

Finally, the new protection uses its own cookie directly, rather than relying
on Django's built-in sessions, so if you're using a different way of storing
user state, such as signed cookies, it will still work.

(If you were an avid user of the old ``CsrfMiddleware``, don't worry; it's been
preserved as ``django.middleware.csrf.CsrfResponseMiddleware``, and will stay
around until Django 1.4.)

Gotchas
=======

Any reasonably magic solution has 'gotchas', and even the new CSRF protection
is no exception.

Firstly, due to cookie subdomain rules, the CSRF protection does not protect
you against any malicious subdomains on the same domain as you; they can easily
retrieve the cookie's value, and use it to send requests. The obvious solution
to this is to make sure your subdomains are all trusted, although that can be
more difficult than it initially sounds [#]_.

Secondly, this protection won't work on outgoing AJAX requests, or custom
content renderers. Incoming AJAX requests are not an issue; browsers have a
same-origin policy, and so the CSRF protection won't try to check them (they
can be identified by their headers).

However, if you're sending HTML out manually as strings, or via a different
template library, you'll need to make sure you manually embed the token
yourself - you can use ``django.middleware.csrf.get_token()`` to get a valid
token to send.

Conclusion
==========

CSRF is a hard problem, and one that isn't fully solved yet in Django (and may
well never be). However, much like auto-escaping, the aim is to give you a
solid, reasonably secure base to work from, and on this front, the new CSRF
protection is a good move forward - it's now up to you to make sure you use it!


.. [#] As Wikipedia puts it, it "exploits the site's trust in the browser". By contrast, XSS explots the browser's trust in the site.
.. [#] This isn't valid HTTP, so please don't send angry letters. I'm trying to illustrate a point here.
.. [#] A lot of people would just close the tab, or browse to another page, or hit back. If there's one thing that web users are good at, it's doing things in a completely illogical manner.
.. [#] CSRF vulnerabilites have *actually* been discovered in online banking interfaces; see Jeff Atwood's article, linked below.
.. [#] Such as click-through attacks, where your site is loaded in a mostly-invisible ``<iframe>``, and a crucial button (such as 'delete') is then placed so you click on it, thinking it's part of the attacking site.
.. [#] If any subdomain allows running arbitary external code in its context, you've got a security hole, for example. This may not seem bad at first, but lots of sites simply embed third-party JavaScript snippets, like Google Analytics, without a second thought.

