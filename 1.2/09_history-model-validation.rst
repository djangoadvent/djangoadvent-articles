:Author:
	Honza Král

###########################
History of Model Validation
###########################

Motivation
==========

When working with Django's awesome (new)forms framework I always felt a slight
pang of pain that something similarly awesome isn't available for Django's
model layer.

Sure, you can create a ModelForm which is very cool and usually good enough
when you are only dealing with user input, especially after newforms admin
landed and enabled people to supply their own form class. But if you are not
dealing with users, don't have a single origin of your model or are dealing
with other developers instead, things can get out of hand pretty quickly. If,
on the other hand, you could define your validation in one place and have it
propagated to all necessary places (ModelForm, admin), that's where you can
really be sure that you have done enough to keep your data in a consistent
state.


First draft
===========

I first started working on model-validation during PyCON in 2008. It was my
first python conference and first time meeting other Djangonauts that I
previously only communicated with over email. It was a very stimulating
atmosphere which inspired me to try and tackle this problem.

At the end of the sprint I had an *almost working* prototype of model
validation. Very naive and missing most of the features it has in 1.2 or even
some that ModelForms have in 1.1 (for example ``validate_unique``). I continued
working on it throughout the year, especially during conferences. Right before
1.0 was out of the door I even got into a discussion with some core devs about
how we can merge it. Unfortunately for me, but not for Django, newforms-admin
was ready earlier and had a higher priority. After newforms-admin landed, my
patch lied in ruins -- it was easier to rewrite it from scratch then to try and
apply the model-validation merge.

The came another conference (Djangocon 2008) where I got two legendary core
developers to sit down and spare a few minutes discussing what model-validation
should look like, especially focusing on validators. I tried to incorporate
their ideas and failed horribly, ending up with another *almost working* patch
in desperate need of some attention. Fortunately [#]_ during yet another
conference I got to talking with yet another great Djangonaut of German origin
who inspired me to apply for the GSOC program. After I answered all the "How
come you are still a student?" questions, I got accepted and thus ran out of
excuses why I never finished the patch and became the champion of *almost
working*.


GSOC
====

`Google Summer of Code`_ is a great program by Google to encourage students to
work on open source projects. Since I was (and still am) a student, managed to
write a feature proposal before the deadline (with full four hours to spare)
and showed that I am stubborn enough to potentially finish it, I got accepted
with Joseph Kocherhans as my mentor. I spent the summer working on the patch,
moving a lot of code around -- from formfields to validators and modelfields,
from modelforms to models and, finally, from old patch to new `SVN branch`_.

At the end of the summer I got, what I thought was, a clean implementation of
model validation. However, as I thought about it -- it was very hard to create
a model that would pass the validation and then fail to save itself into the
database.  In my eagerness to finish the patch and provide a clean solution I
forgot one *minor* detail -- people don't work with clean models, most of the
time, when manipulating a model instance, you work with a partial model, often
constructed by a ModelField with fields excluded.


Back to reality
===============

It became apparent when Joseph merged the branch into trunk that this oversight
on my part is something that will upset many people and, as such, is not
acceptable to be a part of the next release. After a brief panic attack we dove
into the code rethinking our approach to the patch, especially regarding
``ModelForms``.  Where our priority originally was to create a bullet-proof
system that would make it very hard for people to corrupt their models, our new
priority became to keep everything as before the merge and provide a clear
migration path for people wanting to do more validation.

So, essentially after the fourth rethinking of our approach, we arrived at the
API you see today in trunk, which you can read about in the `Validators`_ and
the `Validating Objects`_ documentation.


.. _`Google Summer of Code`: http://code.google.com/soc/
.. _`SVN branch`: http://code.djangoproject.com/browser/django/branches/soc2009/model-validation
.. _`Validators`: http://docs.djangoproject.com/en/dev/ref/validators/#ref-validators
.. _`Validating Objects`: http://docs.djangoproject.com/en/dev/ref/models/instances/#validating-objects

.. [#] I would like to think that this was fortunate for both Django and myself.
