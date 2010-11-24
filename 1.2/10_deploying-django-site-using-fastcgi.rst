:Author:
	Eric Florenzano

=====================================
Deploying a Django Site using FastCGI
=====================================

When developing a site with Django, it's so easy to simply pop open a console
and type:

.. sourcecode:: sh

    python manage.py runserver

With that single management command, our admin media files are served properly,
the Python path is set to properly include our project root, and a simple
autoreloading webserver is started on the port that we specify.  It's all so
easy!

It's no wonder that people are frustrated when it comes to putting their site
into production: there are so many steps in the process, and that makes it
difficult to learn and to get right.  Unsurprisingly, this difficulty has lead
to many articles being written about deploying a Django website.  But almost
all of those articles focus on how to deploy a site using Apache and
`mod_wsgi`_ or `mod_python`_.

There are some times, however, where Apache is not the ideal solution.  Maybe
it's that our `VPS`_ only has 256MB of RAM.  Maybe it's that we want to avoid
the added complexity of Apache in our setup.  Or maybe it's that we simply
don't like Apache.  For these reasons, we might turn our attention to
`FastCGI`_.


First Things First
==================

Before we can get started on doing this deployment, we need to make sure that
we've gotten the base of our system set up.  Firstly, we'll need a server on
which to deploy our app.  This can really be any server and operating system,
but for the sake of simplicity we'll be assuming an Ubuntu-flavored Linux as a
target.

Let's get the basics out of the way, including some compilers, development
headers for Python, and `setuptools`_:

.. sourcecode:: sh

    sudo apt-get install build-essential python-dev python-setuptools

`Nginx`_ should also be installed, as it will act as our webserver.  We can do
this with:

.. sourcecode:: sh

    sudo apt-get install nginx

We should also have `daemontools`_ installed, which is a collection of tools
for managing services.  We're going to be using it to ensure that our service
stays up and running (or at least comes back to life), even in the event of
failure or server restart.  To install daemontools, type:

.. sourcecode:: sh

    sudo apt-get install daemontools

Unfortunately, the package for daemontools in Ubuntu could use a bit of work,
so we have to do a bit more ourselves to finish the installation so that it 
automatically runs at boot time.

If you're running an older version of Ubuntu (before 9.10), you'll want to
create a file at ``/etc/event.d/svscanboot`` with these contents:

.. sourcecode:: sh

    start on runlevel 2
    start on runlevel 3
    start on runlevel 4
    start on runlevel 5
 
    stop on runlevel 0
    stop on runlevel 1
    stop on runlevel 6
 
    respawn
    exec /usr/bin/svscanboot

If you're running a newer installation of Ubuntu, however, you'll want to
create a file at ``/etc/init/svscanboot.conf`` with these contents:

.. sourcecode:: sh

    # svscanboot - Start up daemontools

    description    "Start up daemontools"

    start on runlevel [2345]
    stop on runlevel [!2345]

    respawn

    exec /usr/bin/svscanboot

Now make a directory at ``/etc/service`` by running this command:

.. sourcecode:: sh

    sudo mkdir /etc/service

Finally, we kick off the daemontools process by running this command:

.. sourcecode:: sh

    sudo initctl start svscanboot

We're also going to create a new user for our site:

.. sourcecode:: sh

    adduser mysite

Since we want to be able to use the ``sudo`` command with our user, we'll also
edit ``/etc/sudoers``.  Find the line where it says ``root	ALL=(ALL) ALL``,
and underneath that add the line:

.. sourcecode:: sh

    mysite ALL=(ALL) ALL

Now we'll switch to our new user:

.. sourcecode:: sh

    su - mysite

That's it for the base of our system.  We're purposefully not covering
database, memcached, mail servers, source control, and other various other base
services because they can vary so much depending on personal preference.


Setting up our Python Virtual Environment
=========================================

Now that we have the base of our system installed, we can focus on the fun
stuff.  First we're going to install `virtualenv`_, which is a tool for
creating isolated Python environments.  Then we'll use virtualenv to create an
isolated environment for our app.

.. sourcecode:: sh

    sudo easy_install virtualenv

With our newly-installed copy of virtualenv, we can go ahead and set up a new 
virtual environment:

.. sourcecode:: sh

    mkdir ~/virtualenvs
    virtualenv ~/virtualenvs/mysite

We've created a directory in our home by the name of ``virtualenvs``, and
inside of that we've created a virtualenv called ``mysite``.  Now let's start
using that, and install `pip`_ to make installing packages easier:

.. sourcecode:: sh

    source ~/virtualenvs/mysite/bin/activate
    easy_install pip

Now we need to make sure that we have a package called `Flup`_ installed.  This
package is a set of useful tools for dealing with WSGI applications.  Included
in these tools is an adapter for taking a WSGI application and serving it as
FastCGI (and SCGI and AJP...but that's beyond the scope of this article.)
Django requires this to be installed before you can use the ``runfcgi``
management command.  Using pip we can easily install this:

.. sourcecode:: sh

    pip install flup

If we wanted to use database adapters, imaging libraries, or xml parsers
installed into the system Python path, we would also want to make sure that
those are accessible from our virtualenv by adding a .pth file in the
virtualenv's ``site-packages`` directory:

.. sourcecode:: sh

    echo "/usr/lib/python2.6/dist-packages/" > ~/virtualenvs/mysite/lib/python2.6/site-packages/fix.pth

The next step is to check out our Django code on the server (obviously git can
be replaced with mercurial, svn, or even something like rsync):

.. sourcecode:: sh

    git clone http://github.com/myusername/mysite.git

If your project has a `pip requirements`_ file, you can make use of that now:

.. sourcecode:: sh

    pip install -U -r mysite/requirements.txt
    
Or if you don't have a requirements file, you can install the dependencies
manually as needed (the following is an example):

.. sourcecode:: sh

    pip install -U Django simplejson python-memcached


Choosing options for our FastCGI Server
=======================================

Whew!  We've come a long way in getting our system set up, and we haven't even
gotten to the FastCGI part yet.  Never fear, we're ready to do that now.  Let's
decide on what kinds of options we want to set when we start our FastCGI
server.

The first choice to be made is which concurrency method we want to use:

``threaded``:
    Running a threaded server will run one single process for all of your HTTP requests, which saves a lot on memory, but all the threads are subject to a single `Global Interpreter Lock`_ (GIL).  This means that performance can be hampered on some CPU-intensive loads.  Note that IO takes place outside of the GIL, so IO-intensive loads shouldn't be affected by it.  Also some Python extensions are deemed to not be "thread safe", which means that they cannot be used with this concurrency choice.

``prefork``:
    Running a preforking server will spawn a pool of processes, each with their own separate copy of Django and Python loaded into memory.  This means that it will necessarily use more memory, but it's not subject to the aforementioned problems with the GIL or thread safety.

Let's assume we're interested in FastCGI because we're on a server without a lot 
of memory.  Since the prefork method will use more memory, we'll choose the 
threaded method instead.

Now we get to choose some more options about how the server should act under
load:

``minspare``:
    How many spare processes/threads should the server keep around, at minimum, to be ready and waiting for future requests?

``maxspare``:
    How many spare processes/threads should the server keep around, at maximum, to be ready and waiting for future requests?

``maxrequests`` (prefork only):
    How many requests will each process serve before it's killed and re-created?  To prevent memory leaks from becoming a problem, it's a good idea to set this.

``maxchildren`` (prefork only):
    How many child processes may be handling requests at any given time?

We're running on a small 256MB VPS, so we'll choose some very modest settings
of 2 for `minspare`, 4 for `maxspare`, 6 for `maxchildren`, and 500 for
`maxrequests`.

Finally, we choose our last few settings:

``host``
    What is the hostname on which to listen for incoming connections?

``port``
    On which port should we listen for incoming connections?

``pidfile``
    When the FastCGI server starts up, it will write a file whose contents are a process ID.  This process ID is the `pid`_ of the master thread/process.  This is the process which will handle OS signals like `SIGHUP`_.  This option specifies the location of that file.

After we've made our choices, we can start our server by running the
``runfcgi`` command:

.. sourcecode:: sh

    python manage.py runfcgi method=threaded host=127.0.0.1 port=8080 pidfile=mysite.pid minspare=4 maxspare=30 daemonize=false

Note that we've added a ``daemonize=false`` flag--this should always be set (in
this author's opinion, having a daemonize option at all is a design flaw in the
runfcgi command.)  Also note that running this command will result in a 
``mysite.pid`` file being written out in your project directory, so it's a good
idea to ensure that your source control ignores that file.

Now that we've verified that our FastCGI server starts up properly, let's quit
out and move on to the next step: using daemontools to run this command and
keep it running in the background at all times.


Having Daemontools Run our FastCGI Server
=========================================

Daemontools will look inside all of the subdirectories in the ``/etc/service``
directory, and in each one it will look for an executable file named ``run``.
If it finds one, it will run that executable and restart it if it dies.  So
let's set up a ``mysite`` directory:

.. sourcecode:: sh

    sudo mkdir /etc/service/mysite

Now let's make a little script to run our fastcgi server, use your editor of
choice to write these contents to ``/etc/service/mysite/run``:

.. sourcecode:: sh

    #!/usr/bin/env bash
    
    source /home/mysite/virtualenvs/mysite/bin/activate
    cd /home/mysite/mysite
    exec envuidgid mysite python manage.py runfcgi method=threaded host=127.0.0.1 port=8080 pidfile=mysite.pid minspare=4 maxspare=30 daemonize=false

There's nothing tricky about this--first we make sure we're in the correct
virtualenv, then we change directory to the mysite project directory, and then
we run the ``runfcgi`` command that we discussed earlier.  The ``envuidgid
mysite`` part simply ensures that the following command should be run by the
``mysite`` user instead of root.

The script has to be executable for daemontools to recognize it, so let's make
sure it is:

.. sourcecode:: sh

    sudo chmod +x /etc/service/mysite/run

Now we can verify that it's running by using the ``svstat`` command:

.. sourcecode:: sh

    sudo svstat /etc/service/mysite/

The results should look something like this:

.. sourcecode:: sh

    /etc/service/mysite/: up (pid 3610) 33 seconds

That means that the process is up, it's got a process identifier of 3610, and
it's been up for 33 seconds.  You can use the ``svc`` command to take the
service down like so:

.. sourcecode:: sh

    sudo svc -d /etc/service/mysite/

Then, if you run ``svstat`` on it again, you should get this output:

.. sourcecode:: sh

    /etc/service/mysite/: down 4 seconds, normally up

To bring it back up, simply run this command:

.. sourcecode:: sh

    sudo svc -u /etc/service/mysite/

A full list of ``svc`` commands can be found on `online`_, and is essential
reading if you're going to dive deeper into daemontools.


Configuring Nginx to Talk to our Server
=======================================

We're nearing the finish line, all we have left to do is configure nginx to
talk to our FastCGI server to get its HTTP responses and serve those to the
user.

Ubuntu comes with a helpful file at ``/etc/nginx/fastcgi_params``,
unfortunately it's not quite right.  It encodes a parameter named
``SCRIPT_NAME``, but what our server really wants is ``PATH_INFO``.  You can
either do a search and replace, or copy the contents below into the
``/etc/nginx/fastcgi_params`` file:

.. sourcecode:: sh

    fastcgi_param  QUERY_STRING       $query_string;
    fastcgi_param  REQUEST_METHOD     $request_method;
    fastcgi_param  CONTENT_TYPE       $content_type;
    fastcgi_param  CONTENT_LENGTH     $content_length;

    fastcgi_param  PATH_INFO          $fastcgi_script_name;
    fastcgi_param  REQUEST_URI        $request_uri;
    fastcgi_param  DOCUMENT_URI       $document_uri;
    fastcgi_param  DOCUMENT_ROOT      $document_root;
    fastcgi_param  SERVER_PROTOCOL    $server_protocol;

    fastcgi_param  GATEWAY_INTERFACE  CGI/1.1;
    fastcgi_param  SERVER_SOFTWARE    nginx/$nginx_version;

    fastcgi_param  REMOTE_ADDR        $remote_addr;
    fastcgi_param  REMOTE_PORT        $remote_port;
    fastcgi_param  SERVER_ADDR        $server_addr;
    fastcgi_param  SERVER_PORT        $server_port;
    fastcgi_param  SERVER_NAME        $server_name;

Now we're going to create the definition for our site, let's use our editor and
create the file ``/etc/nginx/sites-available/mysite`` with the following
contents:

.. sourcecode:: nginx

    server {
        listen 80;
        server_name mysite.com www.mysite.com;
        access_log /var/log/nginx/mysite.access.log;

        location /media {
            autoindex on;
            index index.html;
            root /home/mysite/mysite;
            break;
        }
        location / {
            include /etc/nginx/fastcgi_params;
            fastcgi_pass 127.0.0.1:8080;
            break;
        }
    }

This says to listen on port 80 (the standard for HTTP) for *http://mysite.com/*
and *http://www.mysite.com/*.  It says that requests for /media should be served
directly from disk from the ``/home/mysite/mysite/media`` directory.  And most
importantly, it says that anything else should be passed via FastCGI to our
server.

Now let's hook this up via symlink:

.. sourcecode:: sh

    sudo ln -s /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/mysite

And finally we can restart nginx to have the new settings take effect:

.. sourcecode:: sh

    sudo /etc/init.d/nginx restart


Conclusions
===========

We have now set up a very minimal server, using nginx to serve media at blazing
fast speeds, and a pure-python FastCGI server to serve dynamic requests.  Nginx
speaks directly to the FastCGI server without any layers in-between.  Using
daemontools, we have complete control over that FastCGI process, and we can 
stop it, restart it, or change its settings at any time.

The really interesting thing is that with just a few small tweaks, this exact
same stack could be used for a `gunicorn`_, `spawning`_, or `paste`_ solution.
Instead of doing ``fastcgi_pass``, we would simply use ``proxy_pass``.  We'd
still use daemontools to keep the process running and control it.  Almost every
single other step of this article would apply.

This is a very viable alternative to the oft-touted stack of Apache/mod_wsgi
and hopefully after reading this article, more people will consider it as a
deployment method.

.. _`mod_wsgi`: http://code.google.com/p/modwsgi/
.. _`mod_python`: http://www.modpython.org/
.. _`VPS`: http://en.wikipedia.org/wiki/Virtual_private_server
.. _`FastCGI`: http://www.fastcgi.com/drupal/
.. _`setuptools`: http://pypi.python.org/pypi/setuptools
.. _`Nginx`: http://nginx.org/
.. _`daemontools`: http://cr.yp.to/daemontools.html
.. _`virtualenv`: http://pypi.python.org/pypi/virtualenv
.. _`Flup`: http://trac.saddi.com/flup
.. _`pip`: http://pypi.python.org/pypi/pip
.. _`pip requirements`: http://pip.openplans.org/requirement-format.html
.. _`Global Interpreter Lock`: http://wiki.python.org/moin/GlobalInterpreterLock
.. _`pid`: http://en.wikipedia.org/wiki/Process_identifier
.. _`SIGHUP`: http://en.wikipedia.org/wiki/SIGHUP
.. _`online`: http://cr.yp.to/daemontools/svc.html
.. _`gunicorn`: http://github.com/benoitc/gunicorn
.. _`spawning`: http://pypi.python.org/pypi/Spawning
.. _`paste`: http://pythonpaste.org/modules/httpserver
