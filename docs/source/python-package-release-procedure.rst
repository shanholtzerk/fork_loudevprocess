python Package Release Procedures
++++++++++++++++++++++++++++++++++++

release to github
-----------------

-  before updating version

   -  update readme file

.. Padding. See https://github.com/sphinx-doc/sphinx/issues/2258

-  check if update to setup.py needed (for new scripts - 3 places)
-  freeze requirements

.. code-block:: shell

    pip freeze > requirements.txt

-  commit requirements.txt before the following

.. code-block:: shell

   git add requirements.txt
   git commit -m 'package requirements update'

-  update version.py

.. code-block:: shell

    <package>\setup.py install

-  commit change "version x.y.z"

.. code-block:: shell

   git add <package>/version.py
   git commit -m 'version x.y.z'

-  if branch

   -  merge branch to master. see https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging

.. code-block:: shell

      git checkout master
      git merge <branchname>

-  sync master to github (git push)
-  in branch shell

.. code-block:: shell

   git tag x.y.z -m 'version x.y.z'
   git push --tags

-  create documentation

.. code-block:: shell

   git log --oneline 2.0.10..2.0.11 [lastrelease..thisrelease]

-  create documentation (con'd)

   -  on github

      -  click <> code
      -  click releases
      -  click Draft a new release button
      -  reformat output from git log above, into release notes

-  If branch, delete remote and local versions **[best if you wait to do this until after deploy, if a branch was deployed earlier]**

.. code-block:: shell

   -  git push origin --delete <branchname> # delete remote
   -  git branch -d <branchname> # delete local

-
   -  if see the following, try git checkout master at target

      -  [scoretility@sandbox.scoretility.com] out: Your configuration specifies to merge with the ref '<branchname>'
      -  [scoretility@sandbox.scoretility.com] out: from the remote, but no such ref was fetched.


release to PyPi
---------------

test release with editable install
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To test with another package which may be changing

-  see https://pip.pypa.io/en/stable/reference/pip_install/ "Editable Installs"

.. code-block:: shell

    pip uninstall <package>
    pip install -e "C:\Users\lking\Documents\Lou's Software\projects\loutilities\loutilities"

release
~~~~~~~

-  see https://packaging.python.org/tutorials/packaging-projects/

-  for test

   -  set version to x.y.z.\ **devn**

.. code-block:: shell

    python setup.py install sdist bdist_wheel
    twine upload dist/<package>-<version>.*
    # use pypi password

Initial deploy to server
--------------------------
Log into server sudo account

.. code-block:: shell

    ### upload webapp files to target host
    sudo mkdir /var/www/www.<vhost>.com/<repo-name>
    cd /var/www/www.<vhost>.com/<repo-name>
    sudo git clone https://github.com/louking/<repo-name>
    cd /var/www/www.<vhost>.com
    sudo chown -R <vhostuser>:<vhostuser> <repo-name>

    ### Create python virtual environment
    cd /var/www/www.<vhost>.com
    sudo mkdir venv
    sudo chown -R <vhostuser>:<vhostuser> venv
    sudo su <vhostuser>
    python3 -m venv venv
    source venv/bin/activate
    pip install --upgrade pip
    cd /var/www/www.<vhost>.com/<repo-name>/<repo-name>
    pip install -r requirements.txt