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
*   check file opens around csv classes (e.g., DictReader, DictWriter) should use mode `'r'` and `'w'` and `param newline=''`

    *   in fact look for `'wb'` and `'rb'` as there may be other binary file operations which need to be fixed

    *   within `__main__` seems like cannot do `from . import xxx` (e.g., version); needs to be moved to `main()`
        as `from <project> import xxx`
        
*   `argparse.ArgumentParser()` doesn't have version keyword

    *   See https://docs.python.org/3.6/library/argparse.html#action
    *   use

        *   `parser = argparse.ArgumentParser(prog='<package>')`
        *   `parser.add_argument('-v', '--version', action='version', version='%(prog)s {}'.format(__version__))`

*   remove ``venv`` virtualenv for python 2
*   create ``venv`` virtualenv for python 3
*   install requirements (see https://stackoverflow.com/a/49692429/799921)

    .. code-block:: shell

        pip install -r requirements.txt
        pip install pip-upgrader
        pip-upgrade
        pip freeze > requirements.txt

Upgrade to newer version of python 3
----------------------------------------

Development Environment
=========================

* upgrade package requirements (see https://stackoverflow.com/a/49692429/799921)

  .. code-block:: shell

      pip install -U pip
      [pip install pip-upgrader]
      pip-upgrade 

  .. note::
    this updates requirements.txt

* mv ``venv`` virtualenv to ``venv-<oldpyver>`` to save in case of problems with the below
* create ``venv`` virtualenv for new python version, use latest pip

  .. code-block:: shell
      
      python<ver> -m venv venv
        # vscode popup - accept new venv as workspace folder
      venv\scripts\activate
      pip install -U pip

* install requirements

  .. code-block:: shell

      python -m pip cache purge
      pip install -r requirements.txt

* if resolution errors occur remove problem package from requirements.txt and retry to let pip resolve the correct version

* upgrade package requirements again, for new python (see https://stackoverflow.com/a/49692429/799921)

  .. code-block:: shell

      pip-upgrade 
      pip freeze > requirements.txt

* check that pip-upgrade installed compatible versions

  .. code-block:: shell

      pip install -r requirements.txt

* set interpreter for vscode [may have to delete and recreate venv if interpreter cannot be selected]

  * click on python version number in status bar (on bottom of vscode window)
  * make sure it is set to .\\venv\\Scripts\\python.exe
  * restart vscode window

* remove venv-<oldpyver>
* test application, resolve any issues
* commit changes
 
Target Environment
=========================

* stop service

  .. code-block:: shell

    # from sudouser account
    # stop service
    sudo systemctl stop vhost-<appl>-<tld>.service

* upgrade venv

  .. code-block:: shell

    # from <appuser> account (see app configuration)
    (<appuser>) cd /var/www/<tld>.<shortapp>.loutilities.com
    (<appuser>) rm -rf venv/*
    (<appuser>) /usr/local/bin/python<ver> -m venv venv
    (<appuser>) source venv/bin/activate
    (<appuser>/venv) pip install -U pip
    (<appuser>/venv) pip cache purge

* fab deploy release 

  * see :ref:`python-ongoing-development`
  
* install packages and create apache control files

  .. code-block:: shell

    # from sudouser account
    # <portnum> is in /root/bin/mod_wsgi-express-readme.txt
    sudo cat /root/bin/mod_wsgi-express-readme.txt
    # NOTE: for scoretility use sudo /root/bin/init-mod_wsgi-express-uplevel rrwebapp [sandbox.]scoretility.com scoretility scoretility <portnum>
    sudo /root/bin/init-mod_wsgi-express <reponame> <tld>.<shortapp>.loutilities.com <appuser> <appuser> <portnum>
    sudo cp /home/lking/activate-this/activate_this.py /var/www/<tld>.<shortapp>.loutilities.com/venv/bin
    sudo chown -R <appuser>:apache /var/www/<tld>.<shortapp>.loutilities.com/venv/bin/activate_this.py
    # test apache [optional]
    sudo /etc/mod_wsgi-express/<tld>.<shortapp>.loutilities.com/apachectl start
    sudo /etc/mod_wsgi-express/<tld>.<shortapp>.loutilities.com/apachectl stop
    
    # start service
    sudo systemctl start vhost-<appl>-<tld>.service

    # verify vhost operation using browser

  where

  .. code-block:: shell

     <tld> is sandbox, www
     <shortapp> is routes, scores, members, contracts, [others?]
     <reponame> is runningroutes, rrwebapp, members, contracts, [others?]
     <appuser> [check configuration]
