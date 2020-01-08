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

Create VHOST
------------

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
    #  WSGIDaemonProcess www.<vhost>.com user=<vhostuser> group=<vhostuser> threads=3 display-name=%{GROUP}
    #  WSGIScriptAlias / /var/www/www.<vhost>.com/<vhost>/<vhost>/<vhost>.wsgi
    #  WSGIProcessGroup www.<vhost>.com
    #
    #  DocumentRoot /var/www/www.<vhost>.com/<vhost>
    #
    #  <Directory /var/www/www.<vhost>.com/<vhost>>
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

    sudo a2ensite \_default

additional hosts

.. code-block:: shell

    sudo a2ensite <virtualhost>
    sudo apachectl configtest # verify syntax before using
    sudo apachectl restart

Set up VHOST SSL
----------------

.. code-block:: shell

    sudo certbot --apache certonly -d <vhost>
    # maybe like sudo certbot --apache certonly -d www.contracts.loutilities.com -d contracts.loutilities.com
    sudo vim /etc/httpd/sites-available/<vhost>.conf
    #    [uncomment the commented SSL related lines]
    sudo apachectl configtest # verify configuration syntax
    sudo apachectl restart
    sudo certbot renew --dry-run # verify operation
    sudo vim /etc/cron.d/certbot # run twice daily
    #    0 \*/12 \* \* \* root /usr/bin/certbot renew

Archive [ignore]
================

in godaddy (or wherever dns is being hosted), make sure <virtualhost> dns entry points to IP address of this server

.. code-block:: shell

    sudo mkdir -p /var/www/<virtualhost>/

first host on server

.. code-block:: shell

   sudo chmod -R 755 /var/www
   sudo mkdir /etc/httpd/sites-available
   sudo mkdir /etc/httpd/sites-enabled
   # add to end of /etc/httpd/conf/httpd.conf
   #   IncludeOptional sites-enabled/*.conf

create /etc/httpd/sites-available/_default.conf

.. code-block:: apache

   <VirtualHost \*:80>
   DocumentRoot /var/www/html
   </VirtualHost>

create /etc/httpd/sites-available/<virtualhost>.conf

.. code-block:: apache

   <VirtualHost \*:80>
   ServerName <virtualhost>
   WSGIDaemonProcess <appname> user=<webhostuser> group=<webhostuser> threads=5
   WSGIScriptAlias / /var/www/<virtualhost>/<appname>/<appname>.wsgi
   <Directory /var/www/<virtualhost>/<appname> >
   WSGIProcessGroup rrwebapp
   WSGIApplicationGroup %{GLOBAL}
   Order deny,allow
   Allow from all
   AllowOverride All
   </Directory>
   LogLevel warn
   ErrorLog /var/www/<virtualhost>/logs/error.log
   CustomLog /var/www/<virtualhost>/logs/requests.log combined
   </VirtualHost>

(first host on server) sudo a2ensite \_default

.. code-block:: shell

    sudo a2ensite <virtualhost>
    sudo apachectl restart

