Patterns
++++++++++++++++++++++++++++++

Hey, check out https://exploreflask.com/en/latest/index.html which has some patterns which might
be of interest.

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

standalone Editor popup for datatables button handler
-------------------------------------------------------

in beforedatatables.js make declaration for standalone editor and create button handling function

.. code-block:: javascript

    var meeting_invites_editor;

    function meeting_sendinvites(url) {
        fn = function (e, dt, node, config) {
            var that = this;

            // update the url parameter for the create view
            var editorajax = meeting_invites_editor.ajax() || {};
            editorajax.url = url + '?' + setParams(allUrlParams());
            meeting_invites_editor.ajax(editorajax);

            // Ajax request to refresh the data
            $.ajax( {
                // application specific: my application has different urls for different methods
                url: url + '?' + setParams(allUrlParams()),
                type: 'get',
                dataType: 'json',
                success: function ( json ) {
                    // if error, display message - application specific
                    if (json.error) {
                        // this is application specific
                        // not sure if there's a generic way to find the current editor instance
                        meeting_invites_editor.error('ERROR retrieving row from server:<br>' + json.error);

                    } else {
                        // create table from json response. for some reason need dummy div element
                        // else html doesn't have <table> in it
                        var invitestbl = $('<table>')
                        var invites = $('<div>').append(invitestbl)
                        var $th = $('<tr>').append(
                            $('<th>').text('name').attr('align', 'left'),
                            $('<th>').text('email').attr('align', 'left'),
                            $('<th>').text('state').attr('align', 'left'),
                        ).appendTo(invitestbl);
                        $.each(json.invitestates, function(i, invite) {
                            var $tr = $('<tr>').append(
                                $('<td>').text(invite.name),
                                $('<td>').text(invite.email),
                                $('<td>').text(invite.state),
                            ).appendTo(invitestbl);
                        });

                        meeting_invites_editor
                            .title('Send Invitations')
                            .edit(null, false)
                            // no editing id, and don't show immediately
                            .set('invitestates', invites.html())
                            .set('from_email', json.from_email)
                            .set('subject', json.subject)
                            .set('message', json.message)
                            .set('options', json.options)
                            .open();
                    }
                }
            } );
        }
        return fn;
    }

in afterdatables.js, create standalone editor

.. code-block:: javascript

        // https://stackoverflow.com/questions/19237235/jquery-button-click-event-not-firing/19237302
        meeting_invites_editor = new $.fn.dataTable.Editor({
            fields: [
                {name: 'invitestates', data: 'invitestates', label: 'Invitation Status', type: 'display',
                    className: 'field_req full block'},
                {name: 'subject', data: 'subject', label: 'Subject', type: 'text', className: 'field_req full block'},
                {name: 'message', data: 'message', label: 'Message', type: 'ckeditorClassic',
                    className: 'field_req full block'},
                {name: 'from_email', data: 'from_email', label: 'From', type: 'text', className: 'field_req full block'},
                {name: 'options', data: 'options', label: '', type: 'checkbox', className: 'full block',
                    options: [
                        {label: 'Request Status Report', value: 'statusreport'},
                        {label: 'Show Action Items', value: 'actionitems'},
                    ],
                    separator: ',',
                }
            ],
        });

        // buttons needs to be set up outside of ajax call (beforedatatables.js meeting_sendinvites()
        // else the button action doesn't fire (see https://stackoverflow.com/a/19237302/799921 for ajax hint)
        meeting_invites_editor
            .buttons([
                {
                    'text': 'Send Invitations',
                    'action': function () {
                        this.submit( null, null, function(data){
                            var that = this;
                        });
                    }
                },
                {
                    'text': 'Cancel',
                    'action': function() {
                        this.close();
                    }
                }
            ])

        // need to redraw after invite submission in case new Attendees row added to table
        meeting_invites_editor.on('submitComplete closed', function(e) {
            _dt_table.draw();
        });

in view that will display standalone editor form, create div with editor fields

.. code-block:: python

    def meeting_pretablehtml():
        pretablehtml = div()
        with pretablehtml:
            # make dom repository for Editor send invites standalone form
            with div(style='display: none;'):
                dd(**{'data-editor-field': 'invitestates'})
                dd(**{'data-editor-field': 'from_email'})
                dd(**{'data-editor-field': 'subject'})
                dd(**{'data-editor-field': 'message'})
                dd(**{'data-editor-field': 'options'})

        return pretablehtml.render()

in CrudApi descended class, declare button

.. code-block:: python

    buttons=lambda: [
        # 'editor' gets eval'd to editor instance
        {'text': 'Send Invites',
         'name': 'send-invites',
         'editor': {'eval': 'meeting_invites_editor'},
         'url': url_for('admin.meetinginvite', interest=g.interest),
         'action': {
             'eval': 'meeting_sendinvites("{}")'.format(rest_url_for('admin.meetinginvite',
                                                                       interest=g.interest))}
         },

you may need an api to handle button submission, e.g.,

.. code-block:: python

    class MeetingInviteApi(MethodView):

        def __init__(self):
            self.roles_accepted = [ROLE_SUPER_ADMIN, ROLE_MEETINGS_ADMIN]

        def permission(self):
            '''
            determine if current user is permitted to use the view
            '''
            # adapted from loutilities.tables.DbCrudApiRolePermissions
            allowed = False

            # must have meeting_id query arg
            if request.args.get('meeting_id', False):
                for role in self.roles_accepted:
                    if current_user.has_role(role):
                        allowed = True
                        break

            return allowed

        def get(self):
            try:
                # verify user can write the data, otherwise abort (adapted from loutilities.tables._editormethod)
                if not self.permission():
                    db.session.rollback()
                    cause = 'operation not permitted for user'
                    return jsonify(error=cause)

                meeting_id = request.args['meeting_id']
                invitestates, invites = get_invites(meeting_id)

                # set defaults
                meeting = Meeting.query.filter_by(id=meeting_id).one()
                from_email = meeting.organizer.email
                subject = '[{} {}] '.format(meeting.purpose, meeting.date)
                message = ''
                # todo: need to tailor when #274 is fixed
                options = 'statusreport,actionitems'

                # if mail has previously been sent, pick up values used prior
                email = Email.query.filter_by(meeting_id=meeting.id, type=MEETING_INVITE_EMAIL).one_or_none()
                if email:
                    from_email = email.from_email
                    subject = email.subject
                    message = email.message
                    options = email.options

                return jsonify(from_email=from_email, subject=subject, message=message, options=options,
                               invitestates=invitestates)

            except Exception as e:
                exc = ''.join(format_exception_only(type(e), e))
                output_result = {'status': 'fail', 'error': 'exception occurred:\n{}'.format(exc)}
                # roll back database updates and close transaction
                db.session.rollback()
                current_app.logger.error(format_exc())
                return jsonify(output_result)

        def post(self):
            try:
                # verify user can write the data, otherwise abort (adapted from loutilities.tables._editormethod)
                if not self.permission():
                    db.session.rollback()
                    cause = 'operation not permitted for user'
                    return jsonify(error=cause)

                # there should be one 'id' in this form data, 'keyless'
                requestdata = get_request_data(request.form)
                meeting_id = request.args['meeting_id']
                from_email = requestdata['keyless']['from_email']
                subject = requestdata['keyless']['subject']
                message = requestdata['keyless']['message']
                options = requestdata['keyless']['options']

                email = Email.query.filter_by(meeting_id=meeting_id, type=MEETING_INVITE_EMAIL).one_or_none()
                if not email:
                    email = Email(interest=localinterest(), type=MEETING_INVITE_EMAIL, meeting_id=meeting_id)
                    db.session.add(email)

                # save updates, used by generateinvites()
                email.from_email = from_email
                email.subject = subject
                email.message = message
                email.options = options
                db.session.flush()

                agendaitem = generateinvites(meeting_id)

                # use meeting view's dte to get the response data
                thisrow = meeting.dte.get_response_data(agendaitem)
                self._responsedata = [thisrow]

                db.session.commit()
                return jsonify(self._responsedata)

            except Exception as e:
                exc = ''.join(format_exception_only(type(e), e))
                output_result = {'status' : 'fail', 'error': 'exception occurred:\n{}'.format(exc)}
                # roll back database updates and close transaction
                db.session.rollback()
                current_app.logger.error(format_exc())
                return jsonify(output_result)

    bp.add_url_rule('/<interest>/_meetinginvite/rest', view_func=MeetingInviteApi.as_view('meetinginvite'),
                    methods=['GET', 'POST'])

datatable child row
---------------------------------

Adding a child row requires a details-control in the first column, used to expand or contract the row. In the
instantiation of the view

.. code-block:: python

    tableidcontext=lambda row: {
        'rowid': row['rowid'],
    },
    tableidtemplate ='actionitems-{{ rowid }}',
    clientcolumns=[
        # 'data' needs to be '' else get exception converting options from meetings render_template
        # TypeError: '<' not supported between instances of 'str' and 'NoneType' when instantiating with this in child row
        {'data': '',
         'name':'details-control',
         'className': 'details-control shrink-to-fit',
         'orderable': False,
         'defaultContent': '',
         'label': '',
         'type': 'hidden',  # only affects editor modal
         'title': '<i class="fa fa-plus-square" aria-hidden="true"></i>',
         'render': {'eval':'render_plus'},
         },
        :
        ],

The edit button needs to be replaced. This shows the child row edit window underneath the parent row in the table.
In the instantiation of the view

.. code-block:: python

    buttons=[
        :
        'editChildRowRefresh',
        :
    ],

define the layout of the child row using nunjunks template

.. code-block:: jinja

    {% extends "child-row-base.njk" %}
    {% block displayfields %}
        <div class="DTE_Label">Comments</div>
        <div class="DTE_Field_Input">{{ comments | safe }}</div>
    {% endblock %}

without embedded table(s)
============================

.. code-block:: python

    childrowoptions= {
        'template': 'actionitem-child-row.njk',
        'showeditor': True,
        'group': 'interest',
        'groupselector': '#metanav-select-interest',
        'childelementargs': [],
    },

with embedded table(s)
=======================

in the instantiation of the view, identify the child row options

.. code-block:: python

    childrowoptions= {
        'template': 'motion-child-row.njk',
        'showeditor': True,
        'group': 'interest',
        'groupselector': '#metanav-select-interest',
        'childelementargs': [
            {'name':'motionvotes', 'type':CHILDROW_TYPE_TABLE, 'table':motionvotes,
             'tableidtemplate': 'motionvotes-{{ parentid }}',
             'args':{
                     'buttons': ['create', 'editRefresh', 'remove'],
                     'columns': {
                         'datatable': {
                             # uses data field as key
                             'date': {'visible': False}, 'motion': {'visible': False},
                         },
                         'editor': {
                             # uses name field as key
                             'date': {'type': 'hidden'}, 'motion': {'type': 'hidden'},
                         },
                     },
                     'inline' : {
                         # uses name field as key; value is used for editor.inline() options
                         'vote': {'submitOnBlur': True}
                     },
                     'updatedtopts': {
                         'dom': 'frt',
                         'paging': False,
                     },
                 }
             },
        ],
    },

if there are tables in the child row, Editor response data needs tables attribute for each row

.. code-block:: python

    class MotionsView(DbCrudApiInterestsRolePermissions):
        def postprocessrows(self, rows):
            for row in rows:
                context = {
                    'meeting_id': request.args['meeting_id'],
                    'agendaitem_id': row['rowid'],
                }
                tableidcontext {
                    'rowid': row['rowid']
                }

                tablename = 'actionitems'
                tables = [
                    {
                        'name': tablename,
                        'label': 'Action Items',
                        'url': rest_url_for('admin.actionitems', interest=g.interest, urlargs=context),
                        'createfieldvals': context,
                        'tableid': self.childtables[tablename]['table'].tableid(**tableidcontext)
                    }]

                row['tables'] = tables

        def editor_method_postcommit(self, form):
            # this is here in case tables changed during edit action
            self.postprocessrows(self._responsedata)

        def open(self):
            super().open()
            self.postprocessrows(self.output_result['data'])

data dependent columns
---------------------------------

Occasionally it might be useful to determine which columns are included in the view, e.g., based on specifics of the
request, user roles, etc.

In the view class, add code similar to the following

.. code-block:: python

    def check_superadmin(self, col):
        '''
        check if col should be included in display based on user's roles

        :param col: column to check
        :return: True if column should be included
        '''
        rv = True
        if not current_user.has_role(ROLE_SUPER_ADMIN):
            supercols = ['interests', 'last_login_at', 'last_login_ip', 'current_login_ip', 'login_count']
            colname = col['name'].split('.')[0]
            if colname in supercols:
                rv = False
        return rv

    def getdtoptions(self):
        '''limit columns to those this user is allowed to see'''
        dtoptions = super().getdtoptions()
        dtoptions['columns'] = [c for c in dtoptions['columns'] if self.check_superadmin(c)]
        return dtoptions

    def getedoptions(self):
        '''limit form fields to those this user is allowed to see'''
        edoptions = super().getedoptions()
        edoptions['fields'] = [c for c in edoptions['fields'] if self.check_superadmin(c)]
        return edoptions

data dependent select options
---------------------------------

Occasionally it might be useful to determine which select options are included in the select, e.g., based on specifics
of the request, user roles, etc.

Add a class for managing the select options

.. code-block:: python

    # this can also be based on DteDbOptionsPickerBase, but this example make use of DteDbRelationship functions
    class RolesPicker(DteDbRelationship):
        '''
        pick Roles, but special processing based on ROLE_SUPER_ADMIN, i.e., if not ROLE_SUPER_ADMIN only present
        roles allowed for this application
        '''

        def __init__(self, **kwargs):
            # the args dict has default values for arguments added by this derived class
            # caller supplied keyword args are used to update these
            # all arguments are made into attributes for self by the inherited class
            args = dict(
                tablemodel=User,
                fieldmodel=Role,
                labelfield='name',
                formfield='roles',
                dbfield='roles',
                uselist=True,
            )
            args.update(kwargs)

            # this initialization needs to be done before checking any self.xxx attributes
            super().__init__(**args)

        def allowed_roles(self):
            # create a copy so we're not messing with Application record, no more can be configured than current users'
            allowed_roles = current_user.roles[:]
            return allowed_roles

        def set(self, formrow):
            '''
            if not ROLE_SUPER_ADMIN merge newly set roles with those user can't see
            '''
            # these are the roles from the form, but limited to allowed_roles if not ROLE_SUPER_ADMIN
            resultroles = super().set(formrow)
            if not current_user.has_role(ROLE_SUPER_ADMIN):
                theuser = User.query.filter_by(email=formrow['email']).one_or_none()
                allowed_roles = self.allowed_roles()
                if theuser:
                    otherroles = [r for r in theuser.roles if r not in allowed_roles]
                    resultroles += otherroles
            return resultroles

        def get(self, dbrow_or_id):
            '''
            if not ROLE_SUPER_ADMIN only return roles allowed for this user
            '''
            rv = super().get(dbrow_or_id)
            rvnames = rv['name'].split(SEPARATOR)
            rvids = rv['id'].split(SEPARATOR)
            if not current_user.has_role(ROLE_SUPER_ADMIN):
                allowed_roles = self.allowed_roles()
                allowed_role_names = [r.name for r in allowed_roles]
                allowed_role_ids = [str(r.id) for r in allowed_roles]
                rv = {
                    'name': SEPARATOR.join([item for item in rvnames if item in allowed_role_names]),
                    'id': SEPARATOR.join([item for item in rvids if item in allowed_role_ids])
                }
            return rv

        def options(self):
            '''limit visible options to what user can see if not ROLE_SUPER_ADMIN'''
            opts = super().options()
            if not current_user.has_role(ROLE_SUPER_ADMIN):
                allowed_roles = self.allowed_roles()
                allowed_role_ids = [r.id for r in allowed_roles]
                opts = [o for o in opts if o['value'] in allowed_role_ids]
            return opts

Use the created options picker class as part of view instantiation

.. code-block:: python

            clientcolumns=[
                {'data': 'roles', 'name': 'roles', 'label': 'Roles',
                 '_treatment': {'relationship': {'optionspicker': RolesPicker()}}
                 },

row reorder control
-------------------------------------------------------
To use a widget to reorder a row, the model for the table needs to have an **order** field

.. code-block:: python

    class MeetingType(Base):
        __tablename__ = 'meetingtype'
        id                  = Column(Integer(), primary_key=True)
        interest_id         = Column(Integer, ForeignKey('localinterest.id'))
        interest            = relationship('LocalInterest', backref=backref('meetingtypes'))

        order               = Column(Integer)
        meetingtype         = Column(Text)
        options             = Column(Text)
        meetingwording      = Column(Text)
        statusreportwording = Column(Text)
        invitewording       = Column(Text)

        version_id = Column(Integer, nullable=False, default=1)
        __mapper_args__ = {
            'version_id_col': version_id
        }

The `createrow()` method should initialize the order field

.. code-block:: python

    class MeetingTypesView(DbCrudApiInterestsRolePermissions):
        def createrow(self, formdata):
            '''
            provide default for order field when row is created
            :param formdata: data from form
            :return: see super().createrow()
            '''
            max = db.session.query(func.max(MeetingType.order)).filter_by(**self.queryparams).filter(*self.queryfilters).one()
            if max[0]:
                formdata['order'] = max[0] + 1
            else:
                formdata['order'] = 1
            output = super().createrow(formdata)
            return output

The column needs to be defined in the tables instantiation of the DbCrudApiInterestsRolePermissions derived
view, and dt options needs to tell the datatable to reorder based on the **order** field, and that the **order**
field is used for reordering.

.. code-block:: python

    meetingtypes_view = MeetingTypesView(
        clientcolumns=[
            {'data': 'order', 'name': 'order', 'label': 'Reorder',
             'type': 'hidden',
             'className': 'reorder shrink-to-fit',
             'render': {'eval': 'render_grip'},
             },
            ...
        ]

        dtoptions={
            'order': [['order:name', 'asc']],
            'rowReorder': {
                'dataSrc': 'order',
                'selector': 'td.reorder',
                'snapX': True,
            },
            ...
        }
    )

add filters to table view
-------------------------------------------------------
To add a filter to a table, the filter needs to be declared in the pretablehtml block. Additionally
the yadcf options need to be created.

.. code-block:: python

    from loutilities.filters import filtercontainerdiv, filterdiv, yadcfoption

    invites_filters = filtercontainerdiv()
    with invites_filters:
        filterdiv('invites-external-filter-date', 'Date')
        filterdiv('invites-external-filter-name', 'Name')
        filterdiv('invites-external-filter-attended', 'Attended')

    invites_yadcf_options = [
        yadcfoption('date:name', 'invites-external-filter-date', 'range_date'),
        yadcfoption('name:name', 'invites-external-filter-name', 'multi_select', placeholder='Select names', width='200px'),
        yadcfoption('attended:name', 'invites-external-filter-attended', 'select', placeholder='Select', width='100px'),
    ]

    invites_view = InvitesView(
        pretablehtml=invites_filters.render(),
        yadcfoptions=invites_yadcf_options,
        :
    )

If any filters need to be persistent (using session or local storage), in `afterdatatables()`
register these and initialize

.. code-block:: javascript

    function afterdatatables() {
        // set up registered filters (id, default for local storage, transient => don't update local storage
        fltr_register('members-external-filter-members', null, true);

        // initialize all the filters
        fltr_init();
    }