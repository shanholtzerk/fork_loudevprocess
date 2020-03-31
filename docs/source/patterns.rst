Patterns
++++++++++++++++++++++++++++++

loutilities.tables-assets
---------------------------------
files in ``loutilities.tables-assets`` are used in conjunction with the classes in ``loutilities.tables.py``.

Within ``assets``

*   include these js files

    .. code-block:: python

        'datatables.js',  # from loutilities
        'datatables.dataRender.ellipsis.js',  # from loutilities
        'editor.buttons.editrefresh.js',  # from loutilities

*   include these css files

    .. code-block:: python

        'datatables.css',  # from loutilities
        'editor.css',  # from loutilities
        'filters.css',  # from loutilities
        'branding.css',  # from loutilities

To use the ``tables-assets`` files, add this snippet to project ``__init__.py`` within ``create_app``, after ``app``
is created.

.. code-block:: python

    from flask import send_from_directory
    from jinja2 import ChoiceLoader, PackageLoader
    import loutilities

    # add loutilities tables-assets for js/css/template loading
    # see https://adambard.com/blog/fresh-flask-setup/
    #    and https://webassets.readthedocs.io/en/latest/environment.html#webassets.env.Environment.load_path
    # loutilities.__file__ is __init__.py file inside loutilities; os.path.split gets package directory
    loutilitiespath = os.path.join(os.path.split(loutilities.__file__)[0], 'tables-assets', 'static')

    @app.route('/loutilities/static/<path:filename>')
    def loutilities_static(filename):
        return send_from_directory(loutilitiespath, filename)

    # bring in js, css assets here, because app needs to be created first
    from .assets import asset_env, asset_bundles
    with app.app_context():
        # js/css files
        asset_env.append_path(app.static_folder)
        asset_env.append_path(loutilitiespath, '/loutilities/static')

        # templates
        loader = ChoiceLoader([
            app.jinja_loader,
            PackageLoader('loutilities', 'tables-assets/templates')
        ])
        app.jinja_loader = loader

    # initialize assets
    asset_env.init_app(app)
    asset_env.register(asset_bundles)

loutilities.user single sign-on
---------------------------------------

application ``model.py`` (this or similar)

.. code-block:: python

    from loutilities.user.model import db, ManageLocalTables

    # copied by update_local_tables
    class LocalUser(Base):
        __tablename__ = 'localuser'
        id                  = Column(Integer(), primary_key=True)
        user_id             = Column(Integer)
        active              = Column(Boolean)

    # note update_local_tables only copies Interests for current application (g.loutility)
    class LocalInterest(Base):
        __tablename__ = 'localinterest'
        id                  = Column(Integer(), primary_key=True)
        interest_id         = Column(Integer)

    def update_local_tables():
        '''
        keep LocalUser table consistent with external db User table
        '''
        # appname needs to match Application.application
        localtables = ManageLocalTables(db, 'members', LocalUser, LocalInterest)
        localtables.update()


application ``views.userrole.userrole.py``

.. code-block:: python

    from loutilities.user.views.userrole import UserView, InterestView
    from ...model import update_local_tables

    class LocalUserView(UserView):
        def editor_method_postcommit(self, form):
            update_local_tables()
    user = LocalUserView()
    user.register()

    class LocalInterestView(InterestView):
        def editor_method_postcommit(self, form):
            update_local_tables()
    interest = LocalInterestView()
    interest.register()


application ``views.userrole.__init__.py``

.. code-block:: python

    from . import userrole

Within application ``__init__.py``

.. code-block:: python

    # activate views
    from .views import userrole as userroleviews
    from loutilities.user.views import bp as userrole
    app.register_blueprint(userrole)

interests task with single sign-on
---------------------------------------

.. code-block:: python

    # interest_id must be included
    tasktype_dbattrs = 'id,interest_id,tasktype,description'.split(',')
    tasktype_formfields = 'rowid,interest_id,tasktype,description'.split(',')
    tasktype_dbmapping = dict(zip(tasktype_dbattrs, tasktype_formfields))
    tasktype_formmapping = dict(zip(tasktype_formfields, tasktype_dbattrs))

    tasktype = DbCrudApiInterestsRolePermissions(
                        # interest items must be included
                        local_interest_model = LocalInterest,
                        endpointvalues={'interest': '<interest>'},
                        rule = '/<interest>/tasktypes',

                        roles_accepted = [ROLE_SUPER_ADMIN, ROLE_LEADERSHIP_ADMIN],
                        app = bp,   # use blueprint instead of app
                        db = db,
                        model = TaskType,
                        version_id_col = 'version_id',  # optimistic concurrency control
                        template = 'datatables.jinja2',
                        pagename = 'Task Types',
                        endpoint = 'admin.tasktypes',
                        dbmapping = tasktype_dbmapping,
                        formmapping = tasktype_formmapping,
                        clientcolumns = [
                            { 'data': 'tasktype', 'name': 'tasktype', 'label': 'Task Type',
                              'className': 'field_req',
                              },
                            { 'data': 'description', 'name': 'description', 'label': 'Description' },
                        ],
                        servercolumns = None,  # not server side
                        idSrc = 'rowid',
                        buttons = ['create', 'editRefresh', 'remove'],
                        dtoptions = {
                                            'scrollCollapse': True,
                                            'scrollX': True,
                                            'scrollXInner': "100%",
                                            'scrollY': True,
                                      },
                        )
    tasktype.register()

optimistic concurrency control for edit window
---------------------------------------------------
For information on optimistic concurrency control see

  * https://docs.sqlalchemy.org/en/13/orm/versioning.html
  * https://en.wikipedia.org/wiki/Optimistic_concurrency_control
  * https://martinfowler.com/eaaCatalog/optimisticOfflineLock.html

``model.py`` must have the following for each table which uses concurrency control

.. code-block:: python

    version_id = Column(Integer, nullable=False, default=1)
    __mapper_args__ = {
        'version_id_col': version_id
    }

``loutilities.tables.DbCrudApi`` instantiation must have ``version_id_col``, e.g.,

.. code-block:: python

    'version_id_col' : 'version_id',