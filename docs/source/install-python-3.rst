Install Python 3
++++++++++++++++++++++++++++++

Centos
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

Windows
-------------

See https://realpython.com/installing-python/#windows

- navigate to https://www.python.org/downloads/windows/
- select/download latest 3.6 release
- run installer

    - **important** You want to be sure to check the box that says Add Python 3.x to PATH as shown to ensure that the
      interpreter will be placed in your execution path.

- click **Install Now**