:Author:
	Chris Beaven

#############
If Gets Smart
#############

Ah, the humble ``{% if %}`` tag. The cornerstone of Django template display
logic.

Every beginner working with Django 1.1 or below will most likely have tripped
over the fact that the ``{% if %}`` template tag only supports basic boolean
logic.

Restricting the tag to only evaluate if a variable is "true" (i.e. exists, is
not empty, and is not a false boolean value) was a purposeful restriction to
the template system to encourage users to separate their presentation and
business logic.

As a weekend project, I challenged myself to write a smarter if tag and then
released it as a `django snippet`_, which quickly gained popularity. About
eight months later, replacing the if tag was proposed for Django 1.2 and
championed by Luke Plant.

.. _django snippet: http://www.djangosnippets.org/snippets/1350/


So what's new, Mr If?
=====================

From Django 1.2, the ``{% if %}`` template tag has been extended to support
basic Pythonic logic, giving you the ability do the following types of things:

.. sourcecode:: html+django

    {% if you.friends.count > 5 %}You're popular!{% endif %}
    {% if country != "NZ" %}Come and visit New Zealand!{% endif %}

You can still use template filters with this new logic. This gives you the
ability for useful statements like ``{% if messages|length > 3 %}...{% endif
%}``.

If not Python
-------------

Remember this isn't Python: you won't have access to the ``None`` builtin or
any python builtin functions.

``{% if movie == None %}`` won't work. But if you need more than ``{% if not
movie %}`` then you're doing it wrong anyway.

Mixing boolean logic
--------------------

The tag also now handles mixed ``and`` and ``or`` boolean logic. But I'd
encourage you to avoid using it regularly - it quickly leads to confusion. For
example:

.. sourcecode:: html+django

    {% if staff or author and not expired %}
        <a href="{{ edit_url }}">Edit this</a>
    {% endif %}

What happens under certain circumstances would be unclear to designers ("if the
user is staff, but has also expired, what happens then?"). It would be much
clearer calculating this in the view and passing through an ``editable``
context variable. Lets come back to this in a moment.

If you need a different order of evaluation than Python's operator precedence,
use multiple ``if`` tags -- using parentheses are not supported.

Missing variables
-----------------

An important consideration is what happens when a context variable does not
exist. It has been left simple: it equates to ``None``.


Digging deep
============

Let's have a look at what's going on behind the scenes. If you're *not* one of
those people that likes to pull apart radios, you can skip this section.

The parser in the snippet was simple; it worked but it wasn't very deep.
However, the only major bug found in the snippet was with boolean operators not
having precidence over comparison:

    ``x or x == 0`` was being parsed as ``(x or x) == 0`` instead of 
    ``x or (x == 0)``.

While this was able to be fixed, I was never really happy with the parser and
Russell Keith-Magee noted this in his review of the submission branch. Luke
Plant followed up with a link to an article on a parser implementation titled
`Simple Top-down Parsing in Python`_ by Fredrik Lundh.

.. _Simple Top-down Parsing in Python: http://effbot.org/zone/simple-top-down-parsing.htm

The snippet still takes a naive approach to parsing complex logical
expressions. In the snippet, "A or B and C" is parsed as "(A or B) and C", not
the pythonic way "A or (B and C)" (giving operator precedence to AND over OR).

This better parser implementation was used for the patch with the benefits of
correctly implementing full operator precedence. It weights operators (starting
from highest precidence) in the same order as Python: 

 * ``or``
 * ``and``
 * ``not``
 * ``in``
 * comparisons (``==``, ``!=``, ``<``, ``>``,``<=``, ``>=``)

"Too much time?" challenge
--------------------------

Read through that `Simple Top-down Parsing in Python`_ article once or twice,
trying to understand exactly what's going on and then... build the parser
yourself.

If you need to, come back to the article again to give yourself some tips. The
challenge is to re-implement the functionality from the overall ideas rather
than the specific implementation given there.

Being a bit of a sadist, I gave myself this challenge after reading the first
paragraph. `Here's my take`__.

.. __: http://gist.github.com/250128


Just because you can...
=======================

There's a time and a place for using the new functionality of the tag. Keep
your presentation and business logic separated. Perhaps that comparison would
be better pushed to your view? Or a model method?

A lot has been written about the importance of logic separation, but here are a
few reasons:

Being nice to designers
-----------------------

Keeping complex logic out of your templates allows your designers to focus on
what they are good at without having to worry about complexities like operator
precedence or advanced boolean logic.

Separation of thought, and providing readable code
--------------------------------------------------

It is easier to read business logic that is not intermingled with presentation
logic. Similarly, it is easier to look at the presentation of a screen or read
html code without having to sift through the database and security logic.


endif
=====

There's only so much I can ramble on about a basic logic tag, so let us finish
here. Thanks to Luke Plant for his hard work in getting this ready for Django
1.2.

Remember, the official documentation covers all new functionality. Here's a
link to the `if tag`_ section for you.

.. _if tag: http://docs.djangoproject.com/en/dev/ref/templates/builtins/#if

