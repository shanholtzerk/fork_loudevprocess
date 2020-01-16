Linux - Random Procedures
++++++++++++++++++++++++++++++

Set Script as Service
----------------------------

See https://www.tecmint.com/create-new-service-units-in-systemd/

Example of wsgi-servers

``/etc/systemd/system/wsgi-servers.service``

.. code-block:: shell

    [Unit]
    Description = start wsgi servers which run from
    After = httpd.service

    [Service]
    ExecStart = /root/bin/wsgi-servers

    [Install]
    WantedBy = multi-user.target

``/root/bin/wsgi-servers``

.. code-block:: shell

    #!/bin/bash
    source /var/www/www.routes.loutilities.com/venv/bin/activate
    mod_wsgi-express start-server --server-name proxysvr.loutilities.com --port 8001 --user routesmgr --group routesmgr /var/www/www.routes.loutilities.com/runningroutes/runningroutes/runningroutes.wsgi --working-directory /var/www/www.routes.loutilities.com/runningroutes/runningroutes/
    deactivate

then

.. code-block:: shell

    sudo systemctl enable wsgi-servers.service
    sudo systemctl start wsgi-servers.service


