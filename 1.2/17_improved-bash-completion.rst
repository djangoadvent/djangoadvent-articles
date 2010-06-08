:Author:
	Arthur Koziel

########################
Improved Bash Completion
########################

Django 1.2 comes with an improved Bash completion script that enables tab
completion for custom management commands and command options.

The main problem with the old Bash completion script was that the completions
were hardcoded. This meant that only built-in management commands (like
`runserver` or `dumpdata`) could be tab completed.  Developers, however,
commonly extend Django through `custom management commands`_, which made the
Bash completion script inconsistent. Tab completion worked for built-in
management commands, but not for custom ones.

The Bash completion script in Django 1.2 changes that. Instead of a hardcoded
list of completions, the Bash script was modified to call a Python function in
Django. This function dynamically generates a list of possible completions
based on the actual project settings.  It will take into account the custom
management commands from applications listed in the `INSTALLED_APPS` setting.

Installation
============

To get Django's Bash completion working, you'll first have to install the
`bash-completion`_ package. It's a set of functions that enhances Bash's,
otherwise very basic, programmable completion. You can install the package
manually (check the "Installation" section of the projects `README file`_) or
through a package manager that is available on your operating system.  MacPorts
and all major Linux distributions have it available in their package
repositories (it is usually called "bash-completion").  Ubuntu already ships
with the package pre-installed.

Next up, you'll have to source the `django_bash_completion` file. It can be
found in the `extras` directory of an SVN checkout.

.. sourcecode:: bash

    source /path/to/django_bash_completion

If you installed Django through pip or easy_install, the `extras` directory
will not be available. In that case, you'll need to `manually download the
file`_.

The script can be sourced automatically at every login by writing the command
in the `~/.bash_profile` file.

Usage
=====

Once installed, Bash will generate completions for the `django-admin.py`,
`django-admin` and `manage.py` commands. To trigger the completion you'll have
to press the tab key twice after the respective command. For `django-admin.py`,
the output should look like this:

.. sourcecode:: bash

    $ django-admin.py <TAB><TAB>
    cleanup           inspectdb         sqlall            startapp
    compilemessages   loaddata          sqlclear          startproject
    createcachetable  makemessages      sqlcustom         syncdb
    dbshell           reset             sqlflush          test
    diffsettings      runfcgi           sqlindexes        testserver
    dumpdata          runserver         sqlinitialdata    validate
    flush             shell             sqlreset          
    help              sql               sqlsequencereset

The same applies to the `manage.py` command of a project.  Note that the
`manage.py` file must be executable for this to work. If this isn't the case,
you can run `chmod u+x manage.py` to make it executable.

.. sourcecode:: bash

    $ ./manage.py <TAB><TAB>
    changepassword    flush             shell             sqlreset
    cleanup           help              sql               sqlsequencereset
    compilemessages   inspectdb         sqlall            startapp
    createcachetable  loaddata          sqlclear          syncdb
    createsuperuser   makemessages      sqlcustom         test
    dbshell           reset             sqlflush          testserver
    diffsettings      runfcgi           sqlindexes        validate
    dumpdata          runserver         sqlinitialdata    

As you might have noticed, there's a small difference between the generated
completions. The `manage.py` completion will take your project settings into
account. In the above example, the `changepassword` and `createsuperuser`
commands were added because of the installed `django.contrib.auth` application.

Another example is the `django-extensions`_ application, which adds some useful
commands to your Django project. Once you've installed it and added
'`django_extensions`' to your `INSTALLED_APPS` setting, the Bash completion
will include the new commands:

.. sourcecode:: bash

    $ ./manage.py <TAB><TAB>
    changepassword          help                    shell_plus
    clean_pyc               inspectdb               show_urls
    cleanup                 loaddata                sql
    compile_pyc             mail_debug              sqlall
    compilemessages         makemessages            sqlclear
    create_app              passwd                  sqlcustom
    create_command          print_user_for_session  sqldiff
    create_jobs             reset                   sqlflush
    createcachetable        reset_db                sqlindexes
    createsuperuser         runfcgi                 sqlinitialdata
    dbshell                 runjob                  sqlreset
    describe_form           runjobs                 sqlsequencereset
    diffsettings            runprofileserver        startapp
    dumpdata                runscript               sync_media_s3
    dumpscript              runserver               syncdata
    export_emails           runserver_plus          syncdb
    flush                   set_fake_emails         test
    generate_secret_key     set_fake_passwords      testserver
    graph_models            shell                   validate

Something that wasn't possible with the old completion script is option
completion. You can press the tab key twice after a command to view all of it's
options. For the `runserver` command, the output should look like this:

.. sourcecode:: bash

    $ ./manage.py runserver <TAB><TAB>
    --adminmedia=  --noreload     --settings=    --verbosity=   
    --help         --pythonpath=  --traceback

The equal sign after an option name indicates that a value must be specified.

When using the option completion you'll probably notice that a space character
will be added after each completed word. This can be a little annoying for
options that require a value, as you'll always have to press the backspace key
to set the cursor to the correct position. Sadly, there's no solution for this.
A Bash completion script is initialized with an option that either outputs a
space character after each completed word, or doesn't. There's no backward
compatible solution to change the option after the script was sourced. Even
with the option disabled, you'd have to manually enter a space character after
each completed word.  Bash 4.0 fixed this by introducing a new '`compopt`'
builtin that allows completion functions to modify completion options
dynamically. But since some operating systems ship with an old Bash version
(yes, I'm looking at you Apple) this feature couldn't be used.

Tip: Enable single-tab completion
=================================

If you have the Readline library installed, you can tell Bash to display the
list of possible completions by pressing the tab key once instead of twice.
This makes the tab completion even more pleasant to use. To change it, write
the following in your `~/.inputrc`:

.. sourcecode:: bash

    # enable single tab completion
    set show-all-if-ambiguous on 

End
===

I hope this article has helped you to set up and understand the Bash completion
script for Django 1.2. Hopefully it'll save you some keystrokes in the future.

.. _`manually download the file`: http://code.djangoproject.com/browser/django/trunk/extras/django_bash_completion
.. _`README file`: http://git.debian.org/?p=bash-completion/bash-completion.git;a=blob;f=README;hb=HEAD#l1
.. _`custom management commands`: http://docs.djangoproject.com/en/dev/howto/custom-management-commands/
.. _`bash-completion`: http://bash-completion.alioth.debian.org/
.. _`django-extensions`: http://pypi.python.org/pypi/django-extensions/

