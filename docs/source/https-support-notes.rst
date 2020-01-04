HTTPS Support Notes
+++++++++++++++++++++

Prepare for SSL Certificate
---------------------------

-  sudo yum install python-certbot-apache

Get certificates from letsencrypt.com
-------------------------------------

-  certbot --apache -d {server}scoretility.com certonly

-  update /etc/httpd/sites-available/{server}scoretility.com.conf

..

   <VirtualHost \*:80>

   ServerName {server}scoretility.com

   Redirect permanent / https://{server}scoretility.com/

   </VirtualHost>

   <VirtualHost \*:443>

   ServerAdmin lking@pobox.com

   ServerName {server}scoretility.com

   [for production

   ServerName www.scoretility.com

   ServerAlias scoretility.com]

   SSLEngine on

   SSLCertificateFile
   /etc/letsencrypt/live/{server}scoretility.com/fullchain.pem

   SSLCertificateKeyFile
   /etc/letsencrypt/live/{server}scoretility.com/privkey.pem

   SSLCertificateChainFile
   /etc/letsencrypt/live/{server}scoretility.com/chain.pem

   DocumentRoot /var/www/{server}scoretility.com/wordpress

   <Directory />

   Options FollowSymLinks

   AllowOverride None

   </Directory>

   <Directory /var/www/{server}scoretility.com/wordpress>

   Options Indexes FollowSymLinks MultiViews

   AllowOverride All

   Order allow,deny

   allow from all

   </Directory>

   LogLevel warn

   ErrorLog /var/www/{server}scoretility.com/logs/error.log

   CustomLog /var/www/{server}scoretility.com/logs/requests.log combined

   </VirtualHost>

-  sudo apachectl restart

Notes
-----

http://www.wpbeginner.com/wp-tutorials/how-to-add-ssl-and-https-in-wordpress/

https://managewp.com/wordpress-ssl-settings-and-how-to-resolve-mixed-content-warnings

mixed content - see
https://developers.google.com/web/fundamentals/security/prevent-mixed-content/fixing-mixed-content

browser cache -
https://codex.wordpress.org/I_Make_Changes_and_Nothing_Happens

android trust -
https://community.letsencrypt.org/t/android-doesnt-trust-the-certificate/16498/2

admin vs. apache user
~~~~~~~~~~~~~~~~~~~~~

Jason Scaroni suggested I have a non-privileged apache user for
scoretility.com and sandbox.steeplechasers.org, and a privileged
(non-root) admin

One possibility is to remove scoretility and sandboxsteeps from wheel
group (i.e., no sudo for these users) and create lking user (e.g.) for
both. lking is administrative user and would be put into wheel

Not sure but it might be a good idea to have separate ssh keys for
scoretility, sandboxsteeps and loutilityadmin

For sandboxsteeps, need apache to be able to write into the document
tree. Not so sure about scoretility, but I think not.

sandboxsteeps should be added to apache group, and default file create
should be sandboxsteeps:apache

Notes

-  add user -
      https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/4/html/Step_by_Step_Guide/s1-starting-create-account.html

   -  useradd <user>

-  add user to group -
      https://www.howtogeek.com/50787/add-a-user-to-a-group-or-second-group-on-linux/

   -  usermod -a -G <group> <user>

-  change primary group for user

   -  usermod -g <group> <user>

-  remove user from group -
      https://unix.stackexchange.com/questions/29570/how-do-i-remove-a-user-from-a-group

   -  gpasswd -d <user> <group>

-  add acl -

   -  http://unix.stackexchange.com/questions/115631/getting-new-files-to-inherit-group-permissions-on-linux

   -  sudo setfacl -Rdm g:<group>:<perms> <dir>

-  remove acl - sudo setfacl -R --remove-all <file>

-  set group for directory

   -  http://stackoverflow.com/questions/1321168/bash-scripting-how-to-set-the-group-that-new-files-will-be-created-with

   -  chmod g+s <directory>

steps
^^^^^

-  sudo gpasswd -d sandboxsteeps wheel # no sudo for you, one year

-  sudo usermod -a -G apache sandboxsteeps # play nice with apache

-  sudo usermod -g apache sandboxsteeps # now apache is primary group

-  sudo usermod -a -G sandboxsteeps sandboxsteeps # add sandboxsteeps
      group

-  sudo chown -R apache:apache
      /var/www/sandbox.steeplechasers.org/wordpress/ # apache group for
      wordpress files

-  sudo chown -R sandboxsteeps:apache
      /var/www/sandbox.steeplechasers.org/wordpress/wp-content/themes/steeps
      # sandboxsteeps owner for steeps theme

-  sudo chmod -R 700 /var/www/sandbox.steeplechasers.org/wordpress/

-  sudo chmod -R g+r-x+X /var/www/sandbox.steeplechasers.org/wordpress/

-  sudo chmod -R g+w
      /var/www/sandbox.steeplechasers.org/wordpress/wp-content/plugins #
      apache needs write access for some directories

-  sudo chmod -R g+w
      /var/www/sandbox.steeplechasers.org/wordpress/wp-content/themes

-  sudo chmod -R g+w
      /var/www/sandbox.steeplechasers.org/wordpress/wp-content/upgrade

-  sudo chmod -R g+w
      /var/www/sandbox.steeplechasers.org/wordpress/wp-content/uploads

-  sudo chmod -R g+w
      /var/www/sandbox.steeplechasers.org/wordpress/wp-content/wflogs #
      wordfence plugin

-  sudo chown -R sandboxsteeps:sandboxsteeps /home/sandboxsteeps

-  sudo chmod -R g+s /home/sandboxsteeps

Security Tips
-------------

-  apache security hardening -
      http://www.anchor.com.au/hosting/dedicated/Security_Hardening_of_an_Apache_Virtual_Host

-  13 security tips - http://www.tecmint.com/apache-security-tips/

-  wordpress -

   -  https://codex.wordpress.org/Hardening_WordPress

   -  https://codex.wordpress.org/Changing_File_Permissions

   -  http://stackoverflow.com/questions/18352682/correct-file-permissions-for-wordpress

-  backups - https://codex.wordpress.org/WordPress_Backups
