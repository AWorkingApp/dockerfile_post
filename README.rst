This is a DRAFT (possibly including untested, yet-to-be released) edits to the blog post published here: https://www.caktusgroup.com/blog/2017/03/14/production-ready-dockerfile-your-python-django-app/

**Have a suggested tweak or fix?** Feel free to submit a PR.


A Production-ready Dockerfile for Your Python/Django App
========================================================

**Update (April 8, 2019):** I updated this post for Python 3.7 and to use the "slim" (Debian-based) Docker image. This version includes a number of other minor improvements, including a less-magical approach to installing (and removing) system dependencies, better-documented uWSGI settings, a list of the requirements and settings you'll need to match this Dockerfile, support for static file serving via uWSGI (including hashed filenames and cache-forever headers), and an accompanying `GitHub repository <https://github.com/caktus/dockerfile_post/>`_.

Docker has matured a lot since it was released. We've been watching it closely at Caktus, and have been thrilled by the adoption -- both by the community and by service providers. As a team of Python and Django developers, we're always searching for best of breed deployment tools. Docker is a clear fit for packaging the underlying code for many projects, including the Python and Django apps we build at Caktus.


Technical Overview
------------------

There are many ways to containerize a Python/Django app, no one of which could be considered "the best." That being said, I think the following approach provides a good balance of simplicity, configurability, and container size. The specific tools I use are: `Docker <https://www.docker.com/>`_ (of course), the `python:3.7-slim <https://hub.docker.com/_/python/>`_ Docker image (based on Debian Stretch), and `uWSGI <https://uwsgi-docs.readthedocs.io/>`_.

In a previous version of this post, I used `Alpine Linux <https://alpinelinux.org/>`_ as the base image for this
Dockerfile. But now, I'm switching to a Debian- and glibc-based image, because I found `an inconvenient workaround <https://github.com/iron-io/dockers/issues/42#issuecomment-290763088>`_ was required for `musl libc <https://www.musl-libc.org/>`_'s ``strftime()`` implementation. The Debian-based "slim" images are still relatively small; the bare-minimum image described below increases from about 170 MB to 227 MB (~33%) when switching from ``python:3.7-alpine`` to ``python:3.7-slim`` (and updating all the corresponding system packages).

There are many WSGI servers available for Python, and we use both Gunicorn and uWSGI at Caktus. A couple of the benefits of uWSGI are that (1) it's almost entirely configurable through environment variables (which fits well with containers), and (2) it includes `native HTTP support <http://uwsgi-docs.readthedocs.io/en/latest/HTTP.html#can-i-use-uwsgi-s-http-capabilities-in-production>`_, which can circumvent the need for a separate HTTP server like Apache or Nginx.


The Dockerfile
--------------

Without further ado, here's a production-ready ``Dockerfile`` you can use as a starting point for your project (it should be added in your top level project directory, next to the ``manage.py`` script provided by your Django project):

.. code-block:: docker

    FROM python:3.7-slim

    # Install packages needed to run your application (not build deps):
    #   mime-support -- for mime types when serving static files
    #   postgresql-client -- for running database commands
    # We need to recreate the /usr/share/man/man{1..8} directories first because
    # they were clobbered by a parent image.
    RUN set -ex \
        && RUN_DEPS=" \
            libpcre3 \
            mime-support \
            postgresql-client \
        " \
        && seq 1 8 | xargs -I{} mkdir -p /usr/share/man/man{} \
        && apt-get update && apt-get install -y --no-install-recommends $RUN_DEPS \
        && rm -rf /var/lib/apt/lists/*

    # Copy in your requirements file
    ADD requirements.txt /requirements.txt

    # OR, if you’re using a directory for your requirements, copy everything (comment out the above and uncomment this if so):
    # ADD requirements /requirements

    # Install build deps, then run `pip install`, then remove unneeded build deps all in a single step.
    # Correct the path to your production requirements file, if needed.
    RUN set -ex \
        && BUILD_DEPS=" \
            build-essential \
            libpcre3-dev \
            libpq-dev \
        " \
        && apt-get update && apt-get install -y --no-install-recommends $BUILD_DEPS \
        && python3.7 -m venv /venv \
        && /venv/bin/pip install -U pip \
        && /venv/bin/pip install --no-cache-dir -r /requirements.txt \
        \
        && apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false $BUILD_DEPS \
        && rm -rf /var/lib/apt/lists/*

    # Copy your application code to the container (make sure you create a .dockerignore file if any large files or directories should be excluded)
    RUN mkdir /code/
    WORKDIR /code/
    ADD . /code/

    # uWSGI will listen on this port
    EXPOSE 8000

    # Add any static environment variables needed by Django or your settings file here:
    ENV DJANGO_SETTINGS_MODULE=my_project.settings.deploy

    # Call collectstatic (customize the following line with the minimal environment variables needed for manage.py to run):
    RUN DATABASE_URL='' /venv/bin/python manage.py collectstatic --noinput

    # Tell uWSGI where to find your wsgi file (change this):
    ENV UWSGI_WSGI_FILE=my_project/wsgi.py

    # Base uWSGI configuration (you shouldn't need to change these):
    ENV UWSGI_VIRTUALENV=/venv UWSGI_HTTP=:8000 UWSGI_MASTER=1 UWSGI_HTTP_AUTO_CHUNKED=1 UWSGI_HTTP_KEEPALIVE=1 UWSGI_UID=1000 UWSGI_GID=2000 UWSGI_LAZY_APPS=1 UWSGI_WSGI_ENV_BEHAVIOR=holy

    # Number of uWSGI workers and threads per worker (customize as needed):
    ENV UWSGI_WORKERS=2 UWSGI_THREADS=4

    # uWSGI static file serving configuration (customize or comment out if not needed):
    ENV UWSGI_STATIC_MAP="/static/=/code/static/" UWSGI_STATIC_EXPIRES_URI="/static/.*\.[a-f0-9]{12,}\.(css|js|png|jpg|jpeg|gif|ico|woff|ttf|otf|svg|scss|map|txt) 315360000"

    # Deny invalid hosts before they get to Django (uncomment and change to your hostname(s)):
    # ENV UWSGI_ROUTE_HOST="^(?!localhost:8000$) break:400"

    # Uncomment after creating your docker-entrypoint.sh
    # ENTRYPOINT ["/code/docker-entrypoint.sh"]

    # Start uWSGI
    CMD ["/venv/bin/uwsgi", "--show-config"]

We extend from the "slim" flavor of the official Docker image for Python 3.7, install a few dependencies for running our application (i.e., that we want to keep in the final version of the image), copy the folder containing our requirements files to the container, and then, in a single line, (a) install the build dependencies needed, (b) ``pip install`` the requirements themselves (edit this line to match the location of your requirements file, if needed), (c) remove the C compiler and any other OS packages no longer needed, and (d) remove the package lists since they're no longer needed. It's important to keep this all on one line so that Docker will cache the entire operation as a single layer.

Next, we copy our application code to the image, set some default environment variables, and run ``collectstatic``. Be sure to change the values for ``DJANGO_SETTINGS_MODULE`` and ``UWSGI_WSGI_FILE`` to the correct paths for your application (note that the former requires a Python package path, while the latter requires a file system path).

A few notes about other aspects of this Dockerfile:

* I only included a minimal set of OS dependencies here. If this is an established production app, you'll most likely need to visit https://packages.debian.org, search for the Debian package names of the OS dependencies you need, including the ``-dev`` supplemental packages as needed, and add them either to ``RUN_DEPS`` or ``BUILD_DEPS`` in your Dockerfile.
* Adding ``--no-cache-dir`` to the ``pip install`` command saves a additional disk space, as this prevents ``pip`` from `caching downloads <https://pip.pypa.io/en/stable/reference/pip_install/#caching>`_ and `caching wheels <https://pip.pypa.io/en/stable/reference/pip_install/#wheel-cache>`_ locally. Since you won't need to install requirements again after the Docker image has been created, this can be added to the ``pip install`` command. Thanks Hemanth Kumar for this tip!
* uWSGI contains a lot of optimizations for running many apps from the same uWSGI process. These optimizations aren't really needed when running a single app in a Docker container, and can `cause issues <https://discuss.newrelic.com/t/newrelic-agent-produces-system-error/43446/2>`_ when used with certain 3rd-party packages. I've added ``UWSGI_LAZY_APPS=1`` and ``UWSGI_WSGI_ENV_BEHAVIOR=holy`` to the uWSGI configuration to provide a more stable uWSGI experience (the latter will be the default in the next uWSGI release).
* The ``UWSGI_HTTP_AUTO_CHUNKED`` and ``UWSGI_HTTP_KEEPALIVE`` options to uWSGI are needed in the event the container will be hosted behind an Amazon Elastic Load Balancer (ELB), because Django doesn't set a valid ``Content-Length`` header by default, unless the ``ConditionalGetMiddleware`` is enabled. See `the note <http://uwsgi-docs.readthedocs.io/en/latest/HTTP.html#can-i-use-uwsgi-s-http-capabilities-in-production>`_ at the end of the uWSGI documentation on HTTP support for further detail.


Requirements and Settings Files
-------------------------------

Production-ready requirements and settings files are outside the scope of this post, but you'll need to include a few things in your requirements file(s), if they're not there already::

    Django>=2.2rc1,<2.3
    uwsgi>=2.0,<2.1
    dj-database-url>=0.5,<0.6
    # Prevent pip from installing the binary wheel for psycopg2; see:
    # http://initd.org/psycopg/docs/install.html#disabling-wheel-packages-for-psycopg-2-7
    psycopg2>=2.7,<2.8 --no-binary psycopg2

I didn't pin these to specific versions here to help future-proof this post somewhat, but you'll likely want to pin these (and other) requirements to specific versions so things don't suddenly start breaking in production. Of course, you don't have to use any of these packages, but you'll need to adjust the corresponding code elsewhere in this post if you don't.

My ``deploy.py`` settings file looks like this:

.. code-block:: python

    import os

    import dj_database_url

    from . import *  # noqa: F403

    # This is NOT a complete production settings file. For more, see:
    # See https://docs.djangoproject.com/en/dev/howto/deployment/checklist/

    DEBUG = False

    ALLOWED_HOSTS = ['localhost']

    DATABASES['default'] = dj_database_url.config(conn_max_age=600)  # noqa: F405

    STATIC_ROOT = os.path.join(BASE_DIR, 'static')  # noqa: F405

    STATICFILES_STORAGE = 'django.contrib.staticfiles.storage.ManifestStaticFilesStorage'

This bears repeating: This is **not** a production-ready settings file, and you should review `the checklist <https://docs.djangoproject.com/en/dev/howto/deployment/checklist/>`_ in the Django docs (and run ``python manage.py check --deploy --settings=my_project.settings.deploy``) to ensure you've properly secured your production settings file.


Building and Testing the Container
----------------------------------

Now that you have the essentials in place, you can build your Docker image locally as follows:

.. code-block:: bash

    docker build -t my-app .

This will go through all the commands in your Dockerfile, and if successful, store an image with your local Docker server that you could then run:

.. code-block:: bash

    docker run -e DATABASE_URL='' -t my-app

This command is merely a smoke test to make sure uWSGI runs, and won't connect to a database or any other external services.


Running Commands During Container Start-Up
------------------------------------------

As a final step, I recommend creating an ``ENTRYPOINT`` script to run commands as needed during container start-up. This will let us accomplish any number of things, such as making sure Postgres is available or running ``migrate`` during container start-up. Save the following to a file named ``docker-entrypoint.sh`` in the same directory as your ``Dockerfile``:

.. code-block:: bash

    #!/bin/sh
    set -e

    until psql $DATABASE_URL -c '\l'; do
        >&2 echo "Postgres is unavailable - sleeping"
        sleep 1
    done

    >&2 echo "Postgres is up - continuing"

    if [ "x$DJANGO_MANAGEPY_MIGRATE" = 'xon' ]; then
        /venv/bin/python manage.py migrate --noinput
    fi

    exec "$@"

Make sure this file is executable, i.e.:

.. code-block:: bash

    chmod a+x docker-entrypoint.sh

Next, uncomment the following line to your ``Dockerfile``, just above the ``CMD`` statement:

.. code-block:: docker

    ENTRYPOINT ["/code/docker-entrypoint.sh"]

This will (a) make sure a database is available (usually only needed when used with Docker Compose) and (b) run outstanding migrations, if any, if the ``DJANGO_MANAGEPY_MIGRATE`` is set to ``on`` in your environment. Even if you add this entrypoint script as-is, you could still choose to run ``migrate`` or ``collectstatic`` in separate steps in your deployment before releasing the new container. The only reason you might not want to do this is if your application is highly sensitive to container start-up time, or if you want to avoid any database calls as the container starts up (e.g., for local testing). If you do rely on these commands being run during container start-up, be sure to set the relevant variables in your container's environment.


Creating a Production-Like Environment Locally with Docker Compose
------------------------------------------------------------------

To run a complete copy of production services locally, you can use `Docker Compose <https://docs.docker.com/compose/>`_. The following ``docker-compose.yml`` will create a barebones, ephemeral, AWS-like container environment with Postgres for testing your production environment locally.

*This is intended for local testing of your production environment only, and will not save data from stateful services like Postgres upon container shutdown.*

.. code-block:: yaml

    version: '2'

    services:
      db:
        environment:
          POSTGRES_DB: app_db
          POSTGRES_USER: app_user
          POSTGRES_PASSWORD: changeme
        restart: always
        image: postgres:11.2
        expose:
          - "5432"
      app:
        environment:
          DATABASE_URL: postgres://app_user:changeme@db/app_db
          DJANGO_MANAGEPY_MIGRATE: "on"
        build:
          context: .
          dockerfile: ./Dockerfile
        links:
          - db:db
        ports:
          - "8000:8000"

Copy this into a file named ``docker-compose.yml`` in the same directory as your ``Dockerfile``, and then run:

.. code-block:: bash

    docker-compose up --build -d

This downloads (or builds) and starts the two containers listed above. You can view output from the containers by running:

.. code-block:: bash

    docker-compose logs

If all services launched successfully, you should now be able to access your application at http://localhost:8000/ in a web browser.

If you need to debug your application container, a handy way to launch an instance it and poke around is:

.. code-block:: bash

    docker-compose run app /bin/bash


Static Files
------------

You may have noticed that we set up static file serving in uWSGI via the ``UWSGI_STATIC_MAP`` and ``UWSGI_STATIC_EXPIRES_URI`` environment variables. If preferred, you can turn this off and use `Django Whitenoise <http://whitenoise.evans.io/en/stable/>`_ or `copy your static files straight to S3 <https://www.caktusgroup.com/blog/2014/11/10/Using-Amazon-S3-to-store-your-Django-sites-static-and-media-files/>`_.


Blocking ``Invalid HTTP_HOST header`` Errors with uWSGI
-------------------------------------------------------

To avoid Django's ``Invalid HTTP_HOST header`` errors (and prevent any such spurious requests from taking up any more CPU cycles than absolutely necessary), you can also configure uWSGI to return an ``HTTP 400`` response immediately without ever invoking your application code. This can be accomplished by uncommenting and customizing the ``UWSGI_ROUTE_HOST`` line in the Dockerfile above.


Summary
-------

That concludes this high-level introduction to containerizing your Python/Django app for hosting on AWS Elastic Beanstalk (EB), Elastic Container Service (ECS), or elsewhere. Each application and Dockerfile will be slightly different, but I hope this provides a good starting point for your containers. Shameless plug: if you're looking for a simple (and at least temporarily free) way to test your Docker containers on AWS using an Elastic Beanstalk Multicontainer Docker environment or the Elastic Container Service, check out Caktus' very own `AWS Web Stacks <https://github.com/caktus/aws-web-stacks>`_. Good luck!
