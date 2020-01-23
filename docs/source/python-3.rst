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

Install on Windows
--------------------

See https://realpython.com/installing-python/#windows

- navigate to https://www.python.org/downloads/windows/
- select/download latest 3.6 release
- run installer

    - **important** You want to be sure to check the box that says Add Python 3.x to PATH as shown to ensure that the
      interpreter will be placed in your execution path.

- click **Install Now**

Convert python 2 to python 3
-----------------------------------

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

