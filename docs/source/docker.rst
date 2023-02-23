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

Reference code at https://github.com/louking/webmodules/tree/docker-skeleton-v1.0

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

https
------------
https://mindsers.blog/post/https-using-nginx-certbot-docker/
