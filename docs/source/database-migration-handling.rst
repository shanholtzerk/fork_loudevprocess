Database Migration Handling
+++++++++++++++++++++++++++++++++

Getting Started with alembic
----------------------------

See https://www.julo.ch/blog/alembic-introduction/

-  from <project> home directory
-  alembic init versioning
-  edit alembic.ini and env.py based on https://github.com/louking/contracts

   -  in env.py don't forget sys.path.append(os.getcwd())

Database Migration
------------------

Database migration is accomplished by using the `alembic
package <https://pypi.python.org/pypi/alembic>`__. See
https://www.julo.ch/blog/alembic-introduction/ and
https://julo.ch/blog/migrating-content-with-alembic for information
about migrating content.

-   Use latest version of alembic

    .. code-block:: shell

       pip install -U alembic
       pip freeze > requirements.txt

-   Update sqlalchemy model in <project>/<model>.py
-   Repeat as necessary, from within virtualenv

    .. code-block:: shell

       cd [directory above the one with alembic.ini]
       alembic -c <project>/alembic.ini revision --autogenerate -m "<comment>"

-   update output file to fill new tables and new fields

    -  save changes off repo, in case alembic revision needs to be repeated

-   load production database to verify changes

    .. code-block:: shell

       alembic -c <project>/alembic.ini upgrade head

-   test changes
-   if database model needs additional updates, revert to previous version of database

    .. code-block:: shell

         alembic -c <project>/alembic.ini downgrade -1

    -   OR

        -  restore previous backup
        -  drop added tables

-   delete latest alembic conversion file

    -  before deleting you might want to save this in an editor buffer

-  commit changes to alembic conversion file -m "database conversion for xxx"

Export Database from MAMP Server
================================

-  use phpMyAdmin
-  Select database
-  Save alembic_version
-  Click Export

   -  custom
   -  check Format-specific options > Object creation options > Add DROP TABLE / VIEW / PROCEDURE / FUNCTION / EVENT / TRIGGER statement
