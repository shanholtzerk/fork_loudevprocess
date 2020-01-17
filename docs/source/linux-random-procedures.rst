Linux - Random Procedures
++++++++++++++++++++++++++++++

.. _set-script-as-service:

Set Script as Service
----------------------------

See https://www.tecmint.com/create-new-service-units-in-systemd/

Example of ``start-wsgi-server`` usage

``/etc/systemd/system/vhost-<servertype>-<servergroup>.service``

where:

    <servertype>
        one of www, sandbox, beta

    <servergroup>
        like routetility, contractility

contents

.. code-block:: shell

    Description = start routetility www proxy server
    Wants = httpd.service
    After = httpd.service

    [Service]
    ExecStart = /root/bin/start-wsgi-server <project> <servername> <user> <group> <port>

    [Install]
    WantedBy = multi-user.target

``/root/bin/start-wsgi-server``

.. code-block:: shell

    #!/bin/bash
    if [[ $# -eq 0 ]] ; then
       echo "usage:"
       echo "    start-wsgi-server project servername user group port"
       exit 0
    fi

    source /var/www/$2/venv/bin/activate
    mod_wsgi-express start-server --server-name proxysvr.loutilities.com --port $5 --user $3 --group $4 /var/www/$2/$1/$1/$1.wsgi --working-directory /var/www/$2/$1/$1/
    deactivate


then

.. code-block:: shell

    sudo systemctl enable vhost-<servertype>-<servergroup>.service
    sudo systemctl start vhost-<servertype>-<servergroup>.service


