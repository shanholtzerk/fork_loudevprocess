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
--------------------------
Database migration is accomplished by using the `flask-migration
package <https://pypi.org/project/Flask-Migrate/>`__. See
https://flask-migrate.readthedocs.io/en/latest/# for information
about migrating content.

-   Use latest version of flask-migrate

    .. code-block:: shell

       pip install -U flask-migrate
       pip freeze > requirements.txt

-   create app.py (example members)

    .. code-block:: python

        # standard
        import os.path

        # pypi
        from flask_migrate import Migrate

        # homegrown
        from members import create_app
        from members.settings import Production
        from members.model import db

        abspath = os.path.abspath(__file__)
        configpath = os.path.join(os.path.dirname(abspath), 'config', 'members.cfg')
        userconfigpath = os.path.join(os.path.dirname(abspath), 'config', 'users.cfg')
        # userconfigpath first so configpath can override
        configfiles = [userconfigpath, configpath]
        app = create_app(Production(configfiles), configfiles)

        migrate = Migrate(app, db)

-   initialize flask-migrate (optional --multidb)

    .. code-block:: shell

        flask db init --multidb

-   Update sqlalchemy model in <project>/<model>.py
-   Repeat as necessary, from within virtualenv

    .. code-block:: shell

       flask db migrate -m "<comment>"

    -   update output file to fill new tables and new fields

        -  save changes off repo, in case migration revision needs to be repeated

    -   upgrade test database to verify changes

        .. code-block:: shell

           flask db upgrade

    -   test changes
    -   if database model needs additional updates, revert to previous version of database

        .. code-block:: shell

             flask db downgrade

        -   OR

            -  restore previous backup
            -  drop added tables

    -   delete latest migration conversion file

        -  before deleting you might want to save this in an editor buffer

    -  commit changes to migration conversion file -m "database conversion for xxx"

Column update during upgrade
================================

insert after `# ### end Alembic commands ###`

.. code-block:: python

    from sqlalchemy.sql import table, column
    from datetime import datetime

    # default position.has_status_report to True
    position = table('position',
                     column('has_status_report', sa.Boolean()))
    op.execute(
        position.update().\
            values({'has_status_report':op.inline_literal(True)})
    )

    statusreport = table('statusreport',
                     column('update_time', sa.DateTime()))
    now = datetime.now()
    op.execute(
        statusreport.update().\
            values({'update_time':now})
    )


Export Database from MAMP Server
================================

-  use phpMyAdmin
-  Select database
-  Save alembic_version
-  Click Export

   -  custom
   -  check Format-specific options > Object creation options > Add DROP TABLE / VIEW / PROCEDURE / FUNCTION / EVENT / TRIGGER statement

Database Migration (bare alembic - early apps)
------------------------------------------------

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

    -   upgrade test database to verify changes

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

