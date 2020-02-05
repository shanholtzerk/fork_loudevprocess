Linux Virtual Host Setup
++++++++++++++++++++++++++++++

See https://www.digitalocean.com/community/tutorials/how-to-set-up-apache-virtual-hosts-on-centos-7

Create user for VHOST update
----------------------------

Create user

.. code-block:: shell

    sudo adduser <vhostuser>
    sudo passwd <vhostuser>
    su - <vhostuser>
    # - update <vhostuser>'s .bashrc (puts time in history output)
    # -  export HISTTIMEFORMAT="%Y-%m-%d %H:%M "
    mkdir .ssh
    chmod 700 .ssh
    touch .ssh/authorized_keys
    chmod 600 .ssh/authorized_keys
    # - copy puttygen public key, paste into .ssh/authorized_keys
    # - create session configuration in putty
    # - set Connection > SSH > Auth > Private key file for authentication
    #   - copy from PuTTYgen window
    # - set Connection > Data > Auto-login username
    # - Session Save

Put <vhostuser> in apache group, access to <vhostuser> group,
/home/<vhostuser> default <vhostuser> group

.. code-block:: shell

    sudo usermod -a -G apache <vhostuser>
    sudo usermod -g apache <vhostuser>
    sudo usermod -a -G <vhostuser> <vhostuser>
    sudo chown -R <vhostuser>:<vhostuser> /home/<vhostuser>
    sudo chmod -R g+s /home/<vhostuser>/

Update DNS (optional)
--------------------------
May need to create the following records in DNS server.

+-----------------+--------------------+--------------------+
| **type**        | **name**           | **value**          |
+-----------------+--------------------+--------------------+
| A               | <subhost>          | <host ip>          |
+-----------------+--------------------+--------------------+
| CNAME           | www.<subhost>      | <subhost>.<host>   |
+-----------------+--------------------+--------------------+
| CNAME           | sandbox.<subhost>  | <subhost>.<host>   |
+-----------------+--------------------+--------------------+

For example, <subhost> = routes, <host> = routes.loutilities.com

.. _create-vhost:

Create VHOST
------------

The following expects ``mod_wsgi-express`` to be running, see :ref:`set-script-as-service` for details.

Create /etc/httpd/sites-available/www.<vhost>.conf

.. code-block:: apache

    <VirtualHost *:80>
      ServerName <vhost>.com
      ServerAlias www.<vhost>.com
    #  Redirect permanent / https://<vhost>.com/
      DocumentRoot /var/www/www.<vhost>.com
      LogLevel warn
      ErrorLog /var/www/www.<vhost>.com/logs/error.log
      CustomLog /var/www/www.<vhost>.com/logs/requests.log combined

      <Directory /var/www/www.<vhost>.com>
        allow from all
        Options +Indexes
      </Directory>
    </VirtualHost>

    #<VirtualHost *:443>
    #  ServerName <vhost>.com
    #  ServerAlias www.<vhost>.com
    #  ServerAdmin lking@pobox.com
    #  SSLEngine on
    #  SSLCertificateFile /etc/letsencrypt/live/www.<vhost>.com/fullchain.pem
    #  SSLCertificateKeyFile /etc/letsencrypt/live/www.<vhost>.com/privkey.pem
    #  SSLCertificateChainFile /etc/letsencrypt/live/www.<vhost>.com/chain.pem
    #
    #  DocumentRoot /var/www/www.<vhost>.com/<repo-name>
    #
    # # wsgi stuff - <wsgi-port> needs to be unique among vhosts
    #  WSGIScriptReloading On
    #  ProxyPass / http://<wsgi-proxy-host>.com:<wsgi-port>/
    #  ProxyPassReverse / http://<wsgi-proxy-host>.com:<wsgi-port>/
    #  RequestHeader set X-Forwarded-Port 443
    #  RequestHeader set X-Forwarded-Scheme https
    #
    #  <Directory /var/www/www.<vhost>.com/<repo-name>>
    #    Options Indexes FollowSymLinks MultiViews
    #    AllowOverride All
    #    Order deny,allow
    #    allow from all
    #  </Directory>
    #
    #  LogLevel warn
    #  ErrorLog /var/www/www.<vhost>.com/logs/error.log
    #  CustomLog /var/www/www.<vhost>.com/logs/requests.log combined
    #
    #</VirtualHost>

Create the directories to hold the vhost on disk

.. code-block:: shell

    sudo mkdir /var/www/www.<vhost>
    sudo mkdir /var/www/www.<vhost>/logs

Enable VHOST
============

(first host on server)

.. code-block:: shell

    sudo a2ensite _default

additional hosts

.. code-block:: shell

    sudo a2ensite <vhost>
    sudo apachectl configtest # verify syntax before using
    sudo apachectl restart

Set up VHOST SSL
----------------

.. code-block:: shell

    sudo certbot --apache certonly -d <vhost>
    # maybe like sudo certbot --apache certonly -d www.<vhost>.com -d <vhost>.com
    sudo vim /etc/httpd/sites-available/<vhost>.conf
    #    [uncomment the commented SSL related lines]
    sudo apachectl configtest # verify configuration syntax
    sudo apachectl restart
    sudo certbot renew --dry-run # verify operation
    sudo vim /etc/cron.d/certbot # run twice daily
    #    0 \*/12 \* \* \* root /usr/bin/certbot renew
