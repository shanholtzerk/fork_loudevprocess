Patterns
++++++++++++++++++++++++++++++

loutilities.tables-assets
---------------------------------
files in ``loutilities.tables-assets`` are used in conjunction with the classes in ``loutilities.tables.py``.

Within ``assets``

*   include these js files

    .. code-block:: python3

        'datatables.js',  # from loutilities
        'datatables.dataRender.ellipsis.js',  # from loutilities
        'editor.buttons.editrefresh.js',  # from loutilities

*   include these css files

    .. code-block:: python3

        'datatables.css',  # from loutilities
        'editor.css',  # from loutilities
        'filters.css',  # from loutilities
        'branding.css',  # from loutilities

To use the ``tables-assets`` files, add this snippet to project ``__init__.py`` within ``create_app``, after ``app``
is created.

.. code-block:: python3

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

.. code-block:: python3

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

.. code-block:: python3

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

.. code-block:: python3

    from . import userrole

Within application ``__init__.py``

.. code-block:: python3

    # activate views
    from .views import userrole as userroleviews
    from loutilities.user.views import bp as userrole
    app.register_blueprint(userrole)

interests task with single sign-on
---------------------------------------

.. code-block:: python3

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

.. code-block:: python3

    version_id = Column(Integer, nullable=False, default=1)
    __mapper_args__ = {
        'version_id_col': version_id
    }

``loutilities.tables.DbCrudApi`` instantiation must have ``version_id_col``, e.g.,

.. code-block:: python3

    'version_id_col' : 'version_id',

show popup after edit update
---------------------------------------------------

In javascript which runs before the datatable is created, make a function which can be executed by editor which
creates a postEdit event handler. The postEdit event handler uses jquery ui dialog for the popup.

.. code-block:: js

    function meeting_sendreminders(ed) {
        fn = function() {
            var that = this;
            that.processing(true);
            ed.one('postEdit', function(e, json, data, id) {
                that.processing(false);
                var message = $('<div>', {title: 'Generated reminders'});
                var popuphtml = $('<ul>').appendTo(message);
                if (json.newinvites.length > 0) {
                    var newinvites = $('<p>', {html: 'new invites sent to'}).appendTo(popuphtml);
                    var newinvitesul = $('<ul>').appendTo(newinvites);
                    for (var i=0; i<json.newinvites.length; i++) {
                        $('<li>', {html: json.newinvites[i]}).appendTo(newinvitesul);
                    }
                }
                if (json.reminded.length > 0) {
                    var reminders = $('<p>', {html: 'reminders sent to'}).appendTo(popuphtml);
                    var remindersul = $('<ul>').appendTo(reminders);
                    for (var i=0; i<json.reminded.length; i++) {
                        $('<li>', {html: json.reminded[i]}).appendTo(remindersul);
                    }
                }
                message.dialog({
                    modal: true,
                    minWidth: 200,
                    height: 'auto',
                    buttons: {
                        OK: function() {
                            $(this).dialog('close');
                        }
                    }
                });
            })
            // selected rows, false means don't display form
            ed.edit({selected:true}, false).submit();
        }
        return fn;
    }

In the put function, create any self.responsekeys which are required by the postEdit handler. In this example,
self.responsekeysp['reminded'] and self.responsekeysp['newinvites'] are added, for multiple ids which may be
selected.

.. code-block:: python3

    @_editormethod(checkaction='edit', formrequest=True)
    def put(self, thisid):
        # allow multirow editing, i.e., to send emails for multiple selected positions
        theseids = thisid.split(',')
        positions = []
        self._responsedata = []
        users = set()
        for id in theseids:
            # try to coerce to int, but ok if not
            try:
                id = int(id)
            except ValueError:
                pass

            # these just satisfy editor -- is this needed?
            thisdata = self._data[id]
            thisrow = self.updaterow(id, thisdata)
            self._responsedata += [thisrow]

            # collect users which hold this position, and positions which have been selected
            position = Position.query.filter_by(id=id).one()
            users |= set(position.users)
            positions.append(position)

        # send reminder email to each user
        self.responsekeys = {'reminded': [], 'newinvites': []}
        for user in users:
            generatereminder(request.args['meeting_id'], user, positions)
            reminder = generatereminder(request.args['meeting_id'], user, positions)
            if reminder:
                self.responsekeys['reminded'].append('{}'.format(user.name))
            else:
                self.responsekeys['newinvites'].append('{}'.format(user.name))

        # do this at the end to pick up invite.lastreminded (updated in generatereminder())
        # note need to flush to pick up any new invites
        db.session.flush()
        for id in theseids:
            thisdata = self._data[id]
            thisrow = self.updaterow(id, thisdata)
            self._responsedata += [thisrow]


When instantiating the instance subclassed from CrudApi, link the button to the javascript function from above

.. code-block:: python3

    buttons=[
        {
            'extend':'edit',
            'editor': {'eval':'editor'},
            'text': 'Send Reminders',
            'action': {'eval':'meeting_sendreminders(editor)'}
        },
        ...
    ],

spoof id for database behavior on composite records
-----------------------------------------------------

Create a spoofing object

.. code-block:: python3

    class TaskMember():
        '''
        allows creation of "taskmember" object to simulate database behavior
        '''
        def __init__(self, **kwargs):
            for key in kwargs:
                setattr(self, key, kwargs[key])

The methods defined below are new or override methods derived from loutilities.tables.CrudApi.

Define new methods to set/get ids in correct format. self.setid() creates composite id for tracking
multiple database records. self.getids() splits out composite id into constituent
record ids.

.. code-block:: python3

    def setid(self, userid, taskid):
        """
        return combined userid, taskid
        :param userid: id for each LocalUser entry
        :param taskid: id for each Task entry
        :return: id
        """
        return ';'.join([str(userid), str(taskid)])

    def getids(self, id):
        """
        return split of id into local user id, task id
        :param id: id for each  entry
        :return: (localuserid, taskid)
        """
        return tuple([int(usertask) for usertask in id.split(';')])

Override open to use spoofing object to create self.rows.

.. code-block:: python3

    def open(self):
        # retrieve member data from localusers
        members = []
        for localuser in LocalUser.query.filter_by(interest=locinterest).all():
            members.append({'localuser':localuser, 'member': User.query.filter_by(id=localuser.user_id).one()})

        tasksmembers = []
        for member in members:
            # collect all the tasks which are referenced by positions and taskgroups for this member
            tasks = get_member_tasks(member['localuser'])

            # create/add taskmember to list for all tasks
            for task in iter(tasks):
                membertaskid = self.setid(member['localuser'].id, task.id)
                taskmember = TaskMember(
                    id=membertaskid,
                    task=task, task_taskgroups=task.taskgroups,
                    member = member['member'],
                    member_positions = member['localuser'].positions,
                )

            tasksmembers.append(taskmember)

        self.rows = iter(tasksmembers)

Manually handle the row update by overriding updaterow.

.. code-block:: python3

    def updaterow(self, thisid, formdata):
        memberid, taskid = self.getids(thisid)
        luser = LocalUser.query.filter_by(id=memberid).one()
        task = Task.query.filter_by(id=taskid).one()

        # make appropriate updates to the constituent records

        member = {'localuser': luser, 'member': User.query.filter_by(id=luser.user_id).one()}

        taskmember = TaskMember(
            id=thisid,
            task=task, task_taskgroups=task.taskgroups,
            member=member['member'],
            member_positions = member['localuser'].positions,
        )

        return self.dte.get_response_data(taskmember)

button icon for table action on row
-----------------------------------------------------

add button to table, but keep it hidden. make sure it has a name (in CrudApi instantiation)

.. code-block:: python3

    buttons = [
            :
        {'extend': 'editChildRowRefresh',
         'name': 'editRefresh',
         'editor':{'eval': 'editor'},
         'className': 'Hidden',
         },

                OR

        {'extend': 'edit',
         'name': 'view-status',
         'text': 'My Status Report',
         'action': {'eval': 'mystatus_statusreport'},
         'className': 'Hidden',
        },

create a column for the button (in CrudApi instantiation)

.. code-block:: python3

    clientcolumns=[
            :
        {'data': '',  # needs to be '' else get exception converting options from meetings render_template
         # TypeError: '<' not supported between instances of 'str' and 'NoneType'
         'name': 'edit-control',
         'className': 'edit-control shrink-to-fit',
         'orderable': False,
         'defaultContent': '',
         'label': '',
         'type': 'hidden',  # only affects editor modal
         'title': 'Edit',
         'render': {'eval': 'render_icon("fas fa-edit")'},
         },

trigger button when icon clicked (javascript after datatables created)

 .. code-block:: javascript

    // if edit-control clicked, trigger button
    onclick_trigger(_dt_table, 'td.edit-control', 'editRefresh');

use css to style icon

.. code-block:: css

    /* edit selection/control management */
    td.edit-control {
        text-align: center;
        cursor: pointer;
        color: forestgreen;
    }
