:Author:
	Florian Apolloner

##################
Object Permissions
##################

Introduction
============

Nearly every web application provides authorization support to limit access to
certain pages to a privileged group of users. Since the patterns for permission
checks are usually the same for every application, it makes sense for Django to
provide an API for it. Even better, Django does it in a clever way, so
developers can use this API in their applications and, if they like, write
backends for the API itself. This way end-users can switch those backends to
perform permission checks against other databases or servers, simply by
choosing another backend in the settings file.

Django has had support for authentication backends for quite some time. In
Django 1.0 support for authorization backends got added in `r6375`_. The code
which previously checked the permission tables was moved from the user model
into the ``ModelBackend`` and the user objects delegated their checks to the
backend.  Django 1.2 adds object permission checks to the existing backends
in `r11807`_.  For now it is really just a foundation -- the default backend
does not support it, nor does the admin application take advantage of object
permissions.  Nevertheless we can easily enhance the admin application to
perform the necessary checks and write our own backend.

Before we start writing our own backend it is important to know which
possibilities do exist. Django supports multiple authentication/authorization
backends at the same time, which allows us to implement the stuff we want
to support and pass the remaining parts (eg. authentication) to default
backends like ``ModelBackend`` and ``RemoteUserBackend``. There are many ways
to write such a backend -- we could use a LDAP server and query it for
permissions or use the database of another application (like Drupal, phpBB,
etc.) -- but to keep this article short and readable we will use a simple Model
and store the data in our own database.

Writing an object aware authorization backend
=============================================

Let's start simple
------------------

Django installs three default permissions for every model: ``add``, ``change``
and ``delete``. In our case only the ``change`` and ``delete`` permissions make
sense, as the ``add`` permission is not related to an object but the model.
Instead we will add a ``view`` permission. We will use the content types
framework to allow greater reuse-ability. Enough of the talking,  start with
the model:

.. sourcecode:: python

    from django.db import models
    from django.contrib.auth.models import User
    from django.contrib.contenttypes models import ContentType

    class ObjectPermission(models.Model):
        user = models.ForeignKey(User)
        can_view = models.BooleanField()
        can_change = models.BooleanField()
        can_delete = models.BooleanField()

        content_type = models.ForeignKey(ContentType)
        object_id = models.PositiveIntegerField()

There should be nothing new about this model, ``__unicode__``, ``Meta`` and
translations are left out for the sake of simplicity. As we used generic
relations we will be able to display this model as inlines in the admin
application everywhere we want permission checks. Before we start writing the
actual admin integration, here is the code for the backend:

.. sourcecode:: python

    from django.conf import settings
    from django.contrib.contenttypes.models import ContentType
    from django.contrib.auth.models import User

    from objperms.models import ObjectPermission

    class ObjectPermBackend(object):
        supports_object_permissions = True
        supports_anonymous_user = True

        def authenticate(self, username, password):
            return None

        def has_perm(self, user_obj, perm, obj=None):
            if not user_obj.is_authenticated():
                user_obj = User.objects.get(pk=settings.ANONYMOUS_USER_ID)

            if obj is None:
                return False

            ct = ContentType.objects.get_for_model(obj)

            try:
                perm = perm.split('.')[-1].split('_')[0]
            except IndexError:
                return False

            p = ObjectPermission.objects.filter(content_type=ct,
                                                object_id=obj.id,
                                                user=user_obj)
            return p.filter(**{'can_%s' % perm: True}).exists()

Now what is new in this snippet? Compared to Django versions before 1.2 we
added an ``obj`` parameter to the ``has_perm`` method and set the
``supports_*`` attributes to ``True``. The ``supports_*`` attributes got added
in 1.2 to allow new backends to advertise their capabilities. These attributes
are needed to allow older backends to work unmodified in Django 1.2. As
mentioned in the `deprecation timeline`_, in Django 1.4 the attributes will
become redundant and backends have to support ``obj`` and anonymous users.
Django currently requires ``authenticate`` to be implemented, so we implement
it and simply return ``None`` as we don't care about authentication. The only
remaining code is the implementation for the ``has_perm`` method. What about
other methods? There are none, at least none which we will need to write: As we
don't support authentication we don't need to implement ``authenticate`` or
``get_user``.  ``has_module_perms`` does not make sense in our case and does
not support the ``obj`` parameter either. We could implement
``get_all_permissions`` or ``get_group_permissions`` but we won't gain anything
from it; neither does Django use them internally nor do we need them in this
example. That said, those last two methods are the only two you might want to
implement (in addition to ``has_perm``) when writing your own backend. The code
above should be pretty straightforward: We ignore permission checks where the
object is ``None`` -- ``ModelBackend`` can take care of them. Then we check if
the user is authenticated and replace the ``AnonymousUser`` instance with a
real user if he is not authenticated. [#]_ After that we extract the permission
name from the ``perm`` parameter; this parameter is partially redundant as it
also contains the name of the model and the application, which we don't need
because the ``obj`` provides the same info already. But the Django admin
application will pass such names in and therefore we will stick to this
convention. [#]_ One important detail about this backend is that we don't
inherit from ``ModelBackend``, which means we will break Django if we use our
backend as the only one, but luckily we can pass a list of backends to
``AUTHENTICATION_BACKENDS`` in ``settings.py``:

.. sourcecode:: python

    AUTHENTICATION_BACKENDS = (
        'django.contrib.auth.backends.ModelBackend',
        'objperms.backend.ObjectPermBackend',
    )

Now, as the needed code is in place, it is a good time to test it: [#]_

.. sourcecode:: pycon

    >>> page = FlatPage.objects.get(pk=1)
    >>> ct = ContentType.objects.get_for_model(page)
    >>> user = User.objects.get(username='apo') # Don't use a superuser here!
    >>> ObjectPermission.objects.create(user=user, can_view=True,
    ...                                 can_change=True, can_delete=False,
    ...                                 content_type=ct, object_id=page.id)
    <ObjectPermission: >
    >>> user.has_perm('flatpages.delete_flatpage', page)
    False
    >>> user.has_perm('flatpages.view_flatpage', page)
    True
    >>> # As mentioned above we could also use:
    >>> user.has_perm('view', page)
    True

Apparently everything is working fine.

Wrapping the admin application
------------------------------

Now that the backend is working it is time to take a look at the admin
application. Luckily there are only 2 methods to provide:
``has_change_permission`` and ``has_delete_permission``, so we will write a
simple mix-in class. If you intend to change more, a subclass of ``ModelAdmin``
from which the other admin classes inherit might be more adequate:

.. sourcecode:: python

    from django.contrib import admin
    from django.contrib.contenttypes.generic import GenericTabularInline
    from django.contrib.flatpages.models import FlatPage
    from django.contrib.flatpages.admin import FlatPageAdmin as FPAdmin

    from objperms.models import ObjectPermission

    class ObjectPermissionInline(GenericTabularInline):
        model = ObjectPermission
        raw_id_fields = ['user']

    class ObjectPermissionMixin(object):
        def has_change_permission(self, request, obj=None):
            opts = self.opts
            return request.user.has_perm(opts.app_label + '.' + opts.get_change_permission(), obj)

        def has_delete_permission(self, request, obj=None):
            opts = self.opts
            return request.user.has_perm(opts.app_label + '.' + opts.get_delete_permission(), obj)

    class FlatPageAdmin(ObjectPermissionMixin, FPAdmin):
        inlines = FPAdmin.inlines + [ObjectPermissionInline]

    admin.site.unregister(FlatPage)
    admin.site.register(FlatPage, FlatPageAdmin)

There is nothing really new in this snippet -- we just override the
``has_perm`` checks to include the object and add an inline to the admin, so
the user can assign permissions while adding a new flatpage. The last thing we
are going to fix is the changelist view, which currently lists every object,
even those we don't have access to. We could modify the ``queryset`` method to
return only the items the user can edit, but this assumes that a database
backend is used and that is not necessarily the case (e.g. LDAP, etc.).
Instead we wrap the change view, catch the ``PermissionDenied`` error, tell the
user what happened and redirect back to the change list:

.. sourcecode:: python

    # This code goes into ``FlatPageAdmin``
    def change_view(self, request, *args, **kwargs):
        try:
            return super(FlatPageAdmin, self).change_view(request, *args, **kwargs)
        except PermissionDenied, e:
            messages.add_message(request, messages.ERROR, u"You don't have the necessary permissions!")
            return HttpResponseRedirect(reverse('admin:flatpages_flatpage_changelist'))

The above code works fine, but the message gets displayed with a green success
icon, which is not really intuitive. For the sake of simplicity, tweaking the
admin template is left as an exercise for the reader.

.. _`r6375`: http://code.djangoproject.com/changeset/6375
.. _`r11807`: http://code.djangoproject.com/changeset/11807
.. _`deprecation timeline`: http://docs.djangoproject.com/en/dev/internals/deprecation/
.. [#] You can specify the id of the user you want to represent anonymous users
   in your settings file using ``ANONYMOUS_USER_ID``.
.. [#] Although the code supports a simpler form containing just the permission
   name too, but that might break other backends relying on that convention.
.. [#] I will use the flatpages application to test them.
