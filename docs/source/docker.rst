Docker
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

Reload Service When Source Changes
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In order to allow the docker service to pick up changes in the source files, the /app folder needs to be bound to the local source files

`docker-compose.dev.yml`

.. code-block:: docker

    app:
      volumes:
        - ./app/src:/app

Use Debugger from vscode
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
          - 5678:5678
        # see https://aka.ms/vscode-docker-python-debug
        command: ["sh", "-c", "pip install debugpy -t /tmp && python /tmp/debugpy --wait-for-client --listen 0.0.0.0:5678 app.py --nothreading --noreload"]

The application is started with docker compose, including these files, as appropriate. E.g., 

.. code-block:: shell

    docker compose -f docker-compose.yml -f docker-compose.dev.yml -f docker-compose.debug.yml up --build -d

Assuming breakpoints are desired and ``docker-compose.debug.yml`` is used, vscode needs to launch accordingly. The following must be added to vscode's ``launch.json``.
Note the port number 5678 matches that which was used within ``docker-compose.debug.yml``. Also note that the removeRoot must match the python version which 
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
                        "remoteRoot": "/usr/local/lib/python3.9/site-packages"
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
                        "remoteRoot": "/usr/local/lib/python3.9/site-packages"
                    },
                    // see https://code.visualstudio.com/docs/editor/variables-reference#_variables-scoped-per-workspace-folder
                    {
                        "localRoot": "${workspaceFolder:loutilities}/loutilities/",
                        "remoteRoot": "/usr/local/lib/python3.9/site-packages/loutilities/"
                    },

                ],
                "justMyCode": false
            },
        ]
    }

and the application is started with docker compose including these files, as appropriate. E.g., 

.. code-block:: shell

    docker compose -f docker-compose.yml -f docker-compose.dev.yml -f docker-compose.loutilities.yml -f docker-compose.debug.yml up --build -d

Restart Service
^^^^^^^^^^^^^^^^^^^

To restart the service, e.g., after changes have been made in initialization code, use ``docker compose <files> restart`` or ``docker compose <files> stop``, 
``docker compose <files> start``. Also see https://www.shellhacks.com/docker-compose-start-stop-restart-build-single-service/

https
------------

The server has apache running natively. We'll proxy via apache, then take care of routing within the docker compose application
with an nginx container.

Apache setup example for production host:

.. code-block:: ApacheConf

    # www.webmodules.loutilities.us

    # portnum needs to be unique among all vhosts

    <VirtualHost *:80>
        ServerName www.webmodules.loutilities.us
        ServerAlias webmodules.loutilities.us
        # comment out when creating certificate
        Redirect permanent / https://www.webmodules.loutilities.us/
    </VirtualHost>


    <VirtualHost *:443>
        ServerAdmin lking@pobox.com
        ServerName www.webmodules.loutilities.us
        ServerAlias webmodules.loutilities.us

        SSLProxyEngine on
        SSLCertificateFile /etc/letsencrypt/live/www.webmodules.loutilities.us/fullchain.pem
        SSLCertificateKeyFile /etc/letsencrypt/live/www.webmodules.loutilities.us/privkey.pem
        Include /etc/letsencrypt/options-ssl-apache.conf
        SSLCertificateChainFile /etc/letsencrypt/live/www.weewx.lousbrews.info/chain.pem

        ProxyPass / http://localhost:8000/
        ProxyPassReverse / http://localhost:8000/
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