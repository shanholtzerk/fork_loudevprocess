Linux Virtual Host Setup
++++++++++++++++++++++++++++++

See https://www.digitalocean.com/community/tutorials/how-to-set-up-apache-virtual-hosts-on-centos-7

Create user for VHOST update
----------------------------

Create user::

    sudo adduser <vhostuser>
    sudo passwd <vhostuser>
    su - <vhostuser>
    mkdir .ssh
    chmod 700 .ssh
    touch .ssh/authorized_keys
    chmod 600 .ssh/authorized_keys
    copy puttygen public key, paste into .ssh/authorized_keys
    create session configuration in putty
    update <vhostuser>’s .bashrc (puts time in history output)
    -  export HISTTIMEFORMAT="%Y-%m-%d %H:%M "

Put <vhostuser> in apache group, access to <vhostuser> group,
/home/<vhostuser> default <vhostuser> group::

    sudo usermod -a -G apache <vhostuser>
    sudo usermod -g apache <vhostuser>
    sudo usermod -a -G <vhostuser> <vhostuser>
    sudo chown -R <vhostuser>:<vhostuser> /home/<vhostuser>
    sudo chmod -R g+s /home/<vhostuser>/

Create VHOST
------------

Create /etc/httpd/sites-available/www.<vhost>.conf::

       <VirtualHost \*:80>
           ServerName <vhost>
           ServerAlias www.<vhost>
           # Redirect permanent / https://<vhost>/
           DocumentRoot /var/www/www.<vhost>
           LogLevel warn
           ErrorLog /var/www/www.<vhost>/logs/error.log
           CustomLog /var/www/www.<vhost>/logs/requests.log combined
           <Directory /var/www/www.<vhost>>
               allow from all
               Options +Indexes
           </Directory>
       </VirtualHost>
       #<VirtualHost \*:443>
       # ServerName <vhost>
       # ServerAlias www.<vhost>
       # ServerAdmin lking@pobox.com
       # SSLEngine on
       # SSLCertificateFile /etc/letsencrypt/live/www.<vhost>/fullchain.pem
       # SSLCertificateKeyFile /etc/letsencrypt/live/www.<vhost>/privkey.pem
       # SSLCertificateChainFile /etc/letsencrypt/live/www.<vhost>/chain.pem
       #
       # WSGIDaemonProcess www.<vhost> user=contractsmgr group=contractsmgr threads=3 display-name=%{GROUP}
       # WSGIScriptAlias / /var/www/www.<vhost>/contracts/contracts/contracts.wsgi
       # WSGIProcessGroup www.<vhost>
       #
       # DocumentRoot /var/www/www.<vhost>/contracts
       #
       # <Directory /var/www/www.<vhost>/contracts>
       # Options Indexes FollowSymLinks MultiViews
       # AllowOverride All
       # Order allow,deny
       # allow from all
       # </Directory>
       #
       # LogLevel warn
       # ErrorLog /var/www/www.<vhost>/logs/error.log
       # CustomLog /var/www/www.<vhost>/logs/requests.log combined
       #
       #</VirtualHost>

    sudo mkdir /var/www/www.<vhost>
    sudo mkdir /var/www/www.<vhost>/logs

Enable VHOST
============

(first host on server)::

    sudo a2ensite \_default

additional hosts::

    sudo a2ensite <virtualhost>
    sudo apachectl configtest # verify syntax before using
    sudo apachectl restart

Set up VHOST SSL
----------------
::

    sudo certbot --apache certonly -d <vhost>
    sudo vim /etc/httpd/sites-available/<vhost>.conf
        [uncomment the commented SSL related lines]
    sudo apachectl configtest # verify configuration syntax
    sudo apachectl restart
    sudo certbot renew --dry-run # verify operation
    sudo vim /etc/cron.d/certbot # run twice daily
        0 \*/12 \* \* \* root /usr/bin/certbot renew

Archive [ignore]
================

in godaddy (or wherever dns is being hosted), make sure <virtualhost> dns entry points to IP address of this server::

    sudo mkdir -p /var/www/<virtualhost>/

first host on server::

   sudo chmod -R 755 /var/www
   sudo mkdir /etc/httpd/sites-available
   sudo mkdir /etc/httpd/sites-enabled
   add to end of /etc/httpd/conf/httpd.conf
   IncludeOptional sites-enabled/*.conf

create /etc/httpd/sites-available/_default.conf::

   <VirtualHost \*:80>
   DocumentRoot /var/www/html
   </VirtualHost>

create /etc/httpd/sites-available/<virtualhost>.conf::

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

(first host on server) sudo a2ensite \_default::

    sudo a2ensite <virtualhost>
    sudo apachectl restart
