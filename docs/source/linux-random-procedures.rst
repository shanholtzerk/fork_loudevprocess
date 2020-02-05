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

Run the following to set up the server. Choose port by reviewing and updating ``/root/bin/mod_wsgi-express-readme.txt``

    .. code-block:: shell

        sudo /root/bin/init-mod_wsgi-express <repo-name> <vhost> <vhostuser> <vhostuser> <port>

    .. note::

        can these be in a single script file for all of the servers? what will happen if servers are set up while
        their services are running? This would replace the ``/root/bin/mod_wsgi-express-readme.txt`` file

Create the service file ``/etc/systemd/system/vhost-<servertype>-<servergroup>.service``

where:

    <servertype>
        one of www, sandbox, beta [beta is being deprecated]

    <servergroup>
        like routetility, contractility

contents

.. code-block:: shell

    [Unit]
    Description = start <servergroup> <servertype> proxy server
    Wants = httpd.service
    After = httpd.service

    [Service]
    Type = forking
    ExecStart = /etc/mod_wsgi-express/<vhost>/apachectl start
    ExecStop = /etc/mod_wsgi-express/<vhost>/apachectl stop

    [Install]
    WantedBy = multi-user.target


then

.. code-block:: shell

    sudo systemctl enable vhost-<servertype>-<servergroup>.service
    sudo systemctl start vhost-<servertype>-<servergroup>.service

finally, update vhost apache conf file per :ref:`create-vhost`. This is the last step to reduce downtime

