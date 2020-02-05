Python 3
++++++++++++++++++++++++++++++

Install on Centos
----------------------------

See https://janikarhunen.fi/how-to-install-python-3-6-1-on-centos-7

.. code-block:: shell

    sudo yum install -y https://centos7.iuscommunity.org/ius-release.rpm
    sudo yum install -y python3
    sudo yum install -y python3-devel
    mkdir testpython
    cd testpython
    python3 -m venv venv
    source venv/bin/activate
    pip install --upgrade pip

Convert production project from python 2 to python 3
=======================================================

.. note::

    python3 can't ``pip install pywin32`` on linux. ``pywin32`` must be removed from ``requirements.txt`` file before
    ``fab ... deploy``

Add the following code to <project>.wsgi file to support the python3 proxy handler

.. code-block:: python

    # see https://flask.palletsprojects.com/en/1.1.x/deploying/wsgi-standalone/#deploying-proxy-setups
    from werkzeug.middleware.proxy_fix import ProxyFix
    application.wsgi_app = ProxyFix(application.wsgi_app, x_proto=1, x_host=1)


Convert venv to python3

.. code-block:: shell

    cd /var/www/<vhost>
    sudo rm -Rf venv
    sudo mkdir venv
    sudo chown <vhostuser>:<vhostuser> venv
    sudo su <vhostuser>
    (<vhostuser>) python3 -m venv venv
    (<vhostuser>) exit
    sudo cp /home/lking/activate-this/activate_this.py venv/bin
    sudo chown -R <vhostuser>:apache venv/bin/activate_this.py
    sudo su <vhostuser>
    (<vhostuser>) source venv/bin/activate
    (<vhostuser>/<vhost>/venv) pip install --upgrade pip
    (development) fab -H <vhost> deploy

Create python3 service - see :ref:`set-script-as-service`

Install on Windows
--------------------

See https://realpython.com/installing-python/#windows

- navigate to https://www.python.org/downloads/windows/
- select/download latest 3.6 release
- run installer

    - **important** You want to be sure to check the box that says Add Python 3.x to PATH as shown to ensure that the
      interpreter will be placed in your execution path.

- click **Install Now**

Convert development project from python 2 to python 3
=======================================================

*   conversion -

    .. note::

        Check directory twice -- be careful not to overwrite venv, python, etc!

    .. code-block:: shell

        python "C:\Users\lking\AppData\Local\Programs\Python\Python36\Tools\scripts\2to3.py" -w -n -f all -f idioms -f set_literal <package>

*   check for integer division ``/``, replace with ``//``
*   look for usages of ``cmp`` in sort, not supported any more
*   ``urlsafe_b64encode`` requires bytes, not str
*   remove ``venv`` virtualenv for python 2
*   create ``venv`` virtualenv for python 3
*   install requirements (see https://stackoverflow.com/a/49692429/799921)

    .. code-block:: shell

        pip install -r requirements.txt
        pip install pip-upgrader
        pip-upgrade
        pip freeze > requirements.txt

