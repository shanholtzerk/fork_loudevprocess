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
