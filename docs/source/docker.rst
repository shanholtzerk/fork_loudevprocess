Docker Development
++++++++++++++++++++++++++++++++

Installation
-----------------------------
Follow instructions in https://docs.docker.com/get-docker/

Useful Links
----------------

* https://docs.docker.com/get-started/07_multi_container/
  

Network Troubleshooting
-------------------------
using https://github.com/nicolaka/netshoot

.. code-block:: shell

    docker run -it --network [network-name] nicolaka/netshoot

MySQL 
-----------

* https://hub.docker.com/_/mysql/

I was seeing some password issues in production, so I'm saving a few links

* https://medium.com/@crmcmullen/how-to-run-mysql-8-0-with-native-password-authentication-502de5bac661
* https://stackoverflow.com/questions/44010575/mysql-access-denied-for-user-userip-address-remote-access-allowed-for-so

CI/CD Workflow
-------------------

* https://docs.docker.com/language/python/configure-ci-cd/
* https://code.visualstudio.com/docs/containers/reference
* https://code.visualstudio.com/docs/containers/debug-python
* https://medium.com/@lassebenninga/how-to-debug-flask-running-in-docker-compose-in-vs-code-ef37f0f516ee

also see

* https://www.docker.com/blog/tag/python-env-series/
* https://docs.docker.com/compose/production/

Compose repository
--------------------

* https://github.com/docker/awesome-compose

strconv.Atoi: parsing "": invalid syntax
---------------------------------------------

* but be careful: 

  * https://stackoverflow.com/questions/73948802/strconv-atoi-parsing-invalid-syntax/74247419#74247419
  * https://github.com/docker/compose-cli/issues/1537

secrets management
--------------------
* https://blog.diogomonica.com//2017/03/27/why-you-shouldnt-use-env-variables-for-secret-data/ 

docker recommends swarm mode

* https://docs.docker.com/engine/swarm/swarm-tutorial/
* https://docs.docker.com/engine/swarm/how-swarm-mode-works/services/
* https://docs.docker.com/engine/swarm/secrets/
* https://earthly.dev/blog/docker-secrets/

security
------------
* https://sysdig.com/blog/dockerfile-best-practices/
  
deploy compose application
----------------------------

From https://docs.docker.com/compose/production/, "You can use Compose to deploy an app to a remote Docker host by 
setting the DOCKER_HOST, DOCKER_TLS_VERIFY, and DOCKER_CERT_PATH environment variables appropriately. 
See also `Compose CLI environment variables <https://docs.docker.com/compose/environment-variables/envvars/>`_."

Ref: https://www.docker.com/blog/how-to-deploy-on-remote-docker-hosts-with-docker-compose/

Reference code at https://github.com/louking/webmodules/tree/docker-skeleton-v1.1

.. code-block:: shell

    docker context create <contextname> --docker "host=ssh://<user>@<remotemachine>"

Make sure identify file is in ssh_config (C:\\Users\\<username>\\\\.ssh\\config) (https://forums.docker.com/t/docker-context-problem/95105/2)

.. code-block:: shell

    Host <remotemachine>
        User <user>
        HostName <remotemachine>
        IdentityFile <sshkey file>

On target machine give user permissions to access docker without using sudo (https://phoenixnap.com/kb/cannot-connect-to-the-docker-daemon-error method 4)

.. code-block:: shell

    sudo usermod -aG docker <user>

Using context on the local machine, build and bring up the compose app on the remote machine

.. code-block:: shell

    docker --context <contextname> compose -f docker-compose.yml -f docker-compose.prod.yml up --build -d

When there is a new release, the application must be brought down before building and bringing it up

.. code-block:: shell

    docker --context <contextname> compose -f docker-compose.yml -f docker-compose.prod.yml down

or 

.. code-block:: shell

    docker --context <contextname> compose down

Debugging a Docker Service Locally
--------------------------------------

.. note::

    This has been made to work in https://github.com/louking/tm-csv-connector.
    If there are any problems below, please use that repo as an example.

Some Required Files
^^^^^^^^^^^^^^^^^^^^

These allow the database to be upgraded before running the app.

``app/src/app-initdb.d/create-database.sh`` (see `create-database.sh <https://github.com/louking/tm-csv-connector/blob/main/app/src/app-initdb.d/create-database.sh>`_)

``app/dbupgrade_and_run.sh``

.. code-block:: shell

    #!/bin/sh

    # NOTE: file end of line characters must be LF, not CRLF (see https://stackoverflow.com/a/58220487/799921)

    # create database if necessary
    while ! ./app-initdb.d/create-database.sh
    do
        sleep 5
    done

    # initial volume create may cause flask db upgrade to fail
    while ! flask db upgrade
    do
        sleep 5
    done
    exec "$@"

``web/nginx-longtimeout.conf``

.. code-block:: nginx

    # see https://serverfault.com/a/777753

    fastcgi_read_timeout 3600;
    proxy_read_timeout 3600;


Launch Debugger from vscode
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In order for vscode to access the service, a Docker compose file similar to the following should be used. This configures nginx to have a long 
timeout to allow time spent with the service halted, starts the service in the debugger, and exposes the port 5678 which matches the 
debugger --listen port. This allows the service to be initialized by docker compose without being started.

``docker-compose.debug.yml``

.. code-block:: docker

    services:
      web:
        volumes:
            - ./web/nginx-longtimeout.conf:/etc/nginx/conf.d/nginx-longtimeout.conf
      app:
        ports:
        - 5000:5000
        - 5678:5678
        environment:
        - FLASK_APP=/app/app.py
        volumes:
        - ./app/src:/app
        command: ["./dbupgrade_and_run.sh", "sh", "-c", "pip install debugpy -t /tmp && python /tmp/debugpy --wait-for-client --listen 0.0.0.0:5678 -m flask run --no-debugger --no-reload --host 0.0.0.0 --port 5000"]


The application is started with docker compose, including these files, as appropriate. E.g., 

.. code-block:: shell

    docker compose -f docker-compose.yml -f docker-compose.debug.yml up --build -d

or alternately in ``tasks.json`` run task ``docker-compose: debug``

.. code-block:: javascript

    "tasks": [
        {
            "label": "build app",
            "type": "shell",
            "dependsOn": [
                "build docs"
            ],
            "command": "docker compose -f docker-compose.yml build",
            "problemMatcher": []
        },
        {
            "type": "docker-compose",
            "label": "docker-compose: debug",
            "dependsOn": [
                "build app"
            ],
            "dockerCompose": {
                "up": {
                  "detached": true,
                  "build": false,
                },
                "files": [
                  "${workspaceFolder}/docker-compose.yml",
                  "${workspaceFolder}/docker-compose.debug.yml"
                ]
          },
        },
    ]


(in ``tasks.json``, for completeness)

.. code-block:: javascript

    "tasks": [
        {
            "type": "docker-compose",
            "label": "docker-compose: up",
            "dependsOn": [
                "build app"
            ],
            "dockerCompose": {
                "up": {
                  "detached": true,
                  "build": false,
                },
                "files": [
                  "${workspaceFolder}/docker-compose.yml",
                ]
          },
        },
        {
            "type": "docker-compose",
            "label": "docker-compose: down",
            // "dependsOn": [
            //     "build app"
            // ],
            "dockerCompose": {
                "down": {
                //   "services": ["app"]
                },
                "files": [
                  "${workspaceFolder}/docker-compose.yml",
                  "${workspaceFolder}/docker-compose.debug.yml"
                ]
          },
        },
    ]

Assuming breakpoints are desired and ``docker-compose.debug.yml`` is used, vscode needs to launch accordingly. The following must be added to vscode's ``launch.json``.
Note the port number 5678 matches that which was used within ``docker-compose.debug.yml``. Also note that the ``remoteRoot`` value must match the python version which 
was used within the service.

``launch.json``

.. code-block:: javascript

    {
        "configurations": [
            // see https://code.visualstudio.com/docs/containers/docker-compose#_python
            {
                "name": "Python: Remote Attach",
                "type": "python",
                "request": "attach",
                "host": "localhost",
                "port": 5678,
                "pathMappings": [
                    {
                        "localRoot": "${workspaceFolder}/app/src",
                        "remoteRoot": "/app"
                    },
                    // allow debugging of pip installed packages
                    {
                        "localRoot": "${workspaceFolder}/.venv/Lib/site-packages",
                        "remoteRoot": "/usr/local/lib/python3.10/site-packages"
                    }
                ],
                "justMyCode": false
            },
        ]
    }

Debug pypi Package Stored Locally
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If a separate package that is normally loaded via pypi is in the workspace, but is being developed along with the main package (e.g., loutilities), an
additional docker compose file and ``launch.json`` configuration is required.

``docker-compose.loutilities.yml``

.. code-block:: docker

    # use editable loutilities
    services:
      app:
        build: 
          args:
            - PYTHON_LIB_VER=${PYTHON_LIB_VER}
        volumes:
          - ..\..\loutilities\loutilities\loutilities:/usr/local/lib/python${PYTHON_LIB_VER}/site-packages/loutilities

``launch.json``

.. code-block:: javascript

    {
        "configurations":  [
            // see https://code.visualstudio.com/docs/containers/docker-compose#_python
            {
                "name": "Python: Remote Attach (loutilities)",
                "type": "python",
                "request": "attach",
                "port": 5678,
                "host": "localhost",
                "pathMappings": [
                    {
                        "localRoot": "${workspaceFolder}/app/src",
                        "remoteRoot": "/app"
                    },
                    // allow debugging of pip installed packages
                    {
                        "localRoot": "${workspaceFolder}/.venv/Lib/site-packages",
                        "remoteRoot": "/usr/local/lib/python3.10/site-packages"
                    },
                    // see https://code.visualstudio.com/docs/editor/variables-reference#_variables-scoped-per-workspace-folder
                    {
                        "localRoot": "${workspaceFolder:loutilities}/loutilities/",
                        "remoteRoot": "/usr/local/lib/python3.10/site-packages/loutilities/"
                    },

                ],
                "justMyCode": false
            },
        ]
    }

and the application is started with docker compose including these files, as appropriate. E.g., 

.. code-block:: shell

    docker compose -f docker-compose.yml -f docker-compose.loutilities.yml -f docker-compose.debug.yml up --build -d

Restart Service
^^^^^^^^^^^^^^^^^^^

To restart the service, e.g., after changes have been made in initialization code, use ``docker compose <files> restart`` or ``docker compose <files> stop``, 
``docker compose <files> start``. Also see https://www.shellhacks.com/docker-compose-start-stop-restart-build-single-service/

.. _docker-apache-conf:

Apache Configuration
-------------------------

The server has apache running natively. We'll proxy via apache, then take care of routing within the docker compose application
with an nginx container.

Apache setup example for production host:

.. code-block:: ApacheConf

    # www.<vhost>

    <VirtualHost *:80>
        ServerName www.<vhost>
        ServerAlias <vhost>
        # comment out when creating certificate
        Redirect permanent / https://www.<vhost>/
    </VirtualHost>


    <VirtualHost *:443>
        ServerAdmin lking@pobox.com
        ServerName www.<vhost>
        ServerAlias <vhost>

        # comment out when creating certificate
        SSLProxyEngine on
        SSLCertificateFile /etc/letsencrypt/live/www.<vhost>/fullchain.pem
        SSLCertificateKeyFile /etc/letsencrypt/live/www.<vhost>/privkey.pem
        Include /etc/letsencrypt/options-ssl-apache.conf
        SSLCertificateChainFile /etc/letsencrypt/live/www.<vhost>/chain.pem

        ProxyPreserveHost on
        ProxyPass / http://localhost:<port>/
        ProxyPassReverse / http://localhost:<port>/
        RequestHeader set X-Forwarded-Port 443
        RequestHeader set X-Forwarded-Scheme https

    </VirtualHost>

flask db migrations
---------------------------
Execute flask db migrations in the development shell container

.. code-block:: shell

    # initialize migrations environment
    docker exec webmodules-shell-1 flask db init --multidb

    # create migration
    docker exec webmodules-shell-1 flask db migrate -m "migration comment"

To add additional database binds to single database, follow 
https://github.com/miguelgrinberg/Flask-Migrate/issues/179#issuecomment-355344826

.. _initial-deploy-docker:

Initial Deploy of Docker Web App to Server
--------------------------------------------

Create apache configuration (e.g., /etc/httpd/sites-available/www.<vhost>.conf), via :ref:`docker-apache-conf`

Set up user account (once per server)

.. code-block:: shell

    (appuser) vim .bashrc
        export HISTTIMEFORMAT="%Y-%m-%d %H:%M "

    (appuser) mkdir .ssh
    (appuser) chmod 700 .ssh
    (appuser) touch .ssh/authorized_keys
    (appuser) chmod 600 .ssh/authorized_keys

    sudo usermod -aG docker appuser

Follow instructions to update DNS (:ref:`update-dns`), and in :ref:`create-vhost` enable VHOST, set up VHOST SSL

Create server base directory

.. code-block:: shell

    sudo mkdir /var/www/www.<vhost>
    sudo mkdir /var/www/www.<vhost>/logs
    sudo a2ensite www.<vhost>
    sudo apachectl configtest # test configuration created above
    sudo apachectl restart

Create git environment

.. code-block:: shell

    sudo mkdir -p /var/www/www.<vhost>/<appname>
    cd /var/www/www.<vhost>/<appname>
    sudo git clone https://github.com/louking/<appname>
    cd /var/www/www.<vhost>
    sudo chown -R appuser:appuser <appname>
    sudo mkdir /var/www/www.<vhost>/applogs
    sudo chown -R appuser:appuser applogs
    sudo mkdir /var/www/www.<vhost>/<appname>/<appname>/config
    sudo mkdir /var/www/www.<vhost>.loutilities.us/<appname>/<appname>/config/db
    sudo chmod 700 /var/www/www.<vhost>.loutilities.us/<appname>/<appname>/config/db
    # create config file(s)
    sudo chown -R appuser:appuser /var/www/www.<vhost>/<appname>/<appname>/config
    # create <appname>.cfg
    # if needed, create users.cfg
    # if needed, create /var/www/www.<vhost>/<appname>/<appname>/.env
    # if needed, create password.txt file(s)

Build application 

.. code-block:: shell

    (appuser) docker compose -f docker-compose.yml -f docker-compose.<qualifier>.yml build

    where:
        <qualifier> is one of prod, sandbox, dev


Start application

.. code-block:: shell

    (appuser) docker compose -f docker-compose.yml -f docker-compose.<qualifier>.yml up -d

    where:
        <qualifier> is one of prod, sandbox, dev

Push to docker hub

.. code-block:: shell

    docker login
    docker compose push


