Linux - Random Procedures
++++++++++++++++++++++++++++++

.. _set-script-as-service:

Set Script as Service
----------------------------

See https://www.tecmint.com/create-new-service-units-in-systemd/

``/root/bin/init-mod_wsgi-express``

.. code-block:: shell

    #!/bin/bash
    if [[ $# -lt 5 ]] ; then
       echo "usage:"
       echo "    init-mod_wsgi-express project servername user group port"
       exit 0
    fi

    source /var/www/$2/venv/bin/activate
    mod_wsgi-express setup-server --server-name proxysvr.loutilities.com --port $5 --user $3 --group $4 /var/www/$2/$1/$1/$1.wsgi --working-directory /var/www/$2/$1/$1/ --server-root /etc/mod_wsgi-express/$2
    deactivate


Example of ``init-mod_wsgi-express`` usage:

Run the following to set up the server

    .. code-block:: shell

        sudo /root/bin/init-mod_wsgi-express runningroutes sandbox.routes.loutilities.com routesmgr routesmgr 8002

Create the service file ``/etc/systemd/system/vhost-<servertype>-<servergroup>.service``

where:

    <servertype>
        one of www, sandbox, beta

    <servergroup>
        like routetility, contractility

contents

.. code-block:: shell

    [Unit]
    Description = start routetility sandbox proxy server
    Wants = httpd.service
    After = httpd.service

    [Service]
    Type = forking
    ExecStart = /etc/mod_wsgi-express/sandbox.routes.loutilities.com/apachectl start
    ExecStop = /etc/mod_wsgi-express/sandbox.routes.loutilities.com/apachectl stop

    [Install]
    WantedBy = multi-user.target


then

.. code-block:: shell

    sudo systemctl enable vhost-<servertype>-<servergroup>.service
    sudo systemctl start vhost-<servertype>-<servergroup>.service


