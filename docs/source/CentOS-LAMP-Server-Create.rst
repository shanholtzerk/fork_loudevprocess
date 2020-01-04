CentOS LAMP Server Create
+++++++++++++++++++++++++++++++

This uses digitalocean for server and volume creation, but most of this
could be used for any CentOS LAMP server.

Create Droplet
==============

-  log into digitalocean.com
-  Create > Droplets <name> 1 GB / 30 GB Disk / Centos 7 / NYC1 IP

Some Upkeep [root]
==================
::

    yum install -y vim
    yum -y update (see /var/log/yum.log for all updates)
    shutdown -r now


Additional repos
----------------
::

    yum install -y epel-release

sudo user
=========

Create non-root user and give sudo access [root]
------------------------------------------------
::

    adduser <user>; passwd <user>
    mkdir /home/<user>/.ssh; cp .ssh/authorized_keys /home/<user>/.ssh
    chown -R <user>:<user> /home/<user>/.ssh
    gpasswd -a <user> wheel

Set sudo timeout (minutes)
--------------------------
::

    sudo visudo # replace 'Defaults env_reset' with following
        Defaults env_reset,timestamp_timeout=30

Set up digitalocean agent for monitoring [root]
===============================================
::

    curl -sSL https://agent.digitalocean.com/install.sh \| sh

history should display date/time
================================
::

    export HISTTIMEFORMAT="%Y-%m-%d %H:%M " # add this to ~/.bashrc, /root/.bashrc

Set local time and keep in sync
===============================
::

    sudo timedatectl set-timezone America/New_York
    sudo yum -y install ntp
    sudo systemctl start ntpd
    sudo systemctl enable ntpd

Turn off cron information to /var/log/messages
==============================================
::

    sudo vim /etc/rc.local # add /usr/bin/systemd-analyze set-log-level notice
    sudo /usr/bin/systemd-analyze set-log-level notice
    sudo chmod +x /etc/rc.d/rc.local

LAMP and Security Stack
=======================

Set up LAMP stack 
------------------

See https://www.digitalocean.com/community/tutorials/how-to-install-linux-apache-mysql-php-lamp-stack-on-centos-7

::

    sudo yum -y install httpd
    sudo systemctl start httpd.service
    sudo systemctl enable httpd.service
    sudo yum -y install mariadb-server mariadb
    sudo systemctl start mariadb
    sudo mysql_secure_installation
    sudo systemctl enable mariadb.service
    sudo yum -y install php php-mysql
    sudo systemctl restart httpd.service
    sudo systemctl start firewalld
    sudo firewall-cmd --permanent --add-service=ssh
    sudo firewall-cmd --permanent --add-service=http
    sudo firewall-cmd --permanent --add-service=https
    sudo firewall-cmd --permanent --add-service=smtp
    sudo firewall-cmd --reload
    sudo systemctl enable firewalld
    sudo yum install -y python-certbot-apache

install additional PHP versions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

See https://stackoverflow.com/a/50079574/799921 and
https://blog.remirepo.net/post/2016/04/16/My-PHP-Workstation
::

    sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm -y
    sudo yum install http://rpms.remirepo.net/enterprise/remi-release-7.rpm -y
    sudo yum install yum-utils -y
    sudo yum install php56 -y
    sudo yum install php73 -y
    sudo yum install php56-php-fpm -y
    sudo yum install php73-php-fpm -y
    sudo systemctl stop php56-php-fpm
    sudo systemctl stop php73-php-fpm
    sudo vim /opt/remi/php56/root/etc/php-fpm.d/www.conf
    -  change listen port to 9056
    sudo vim /etc/opt/remi/php73/php-fpm.d/www.conf
    -  change listen port to 9073
    sudo systemctl start php56-php-fpm
    sudo systemctl enable php56-php-fpm
    sudo yum install php56-php-mysqlnd -y
    sudo systemctl restart php56-php-fpm
    sudo systemctl start php73-php-fpm
    sudo systemctl enable php73-php-fpm
    sudo yum install php73-php-mysqlnd -y
    sudo yum install php56-php-xml -y
    sudo yum install php73-php-xml -y
    sudo systemctl restart php73-php-fpm
    add following to the vhost config file (in /etc/httpd/sites-available)
    -  <FilesMatch \\.php$>
    -  SetHandler "proxy:fcgi://127.0.0.1:9073"
    ..
    [or 9056, based on php version]
    -  </FilesMatch>
    sudo apachectl restart
    sudo systemctl restart php73-php-fpm

Create a2ensite, a2dissite
--------------------------
See http://www.tecmint.com/apache-virtual-hosting-in-centos/
::

   sudo vim /usr/bin/a2ensite
        #!/bin/bash
        if test -d /etc/httpd/sites-available && test -d /etc/httpd/sites-enabled  ; then
        echo "-----------------------------------------------"
        else
        mkdir /etc/httpd/sites-available
        mkdir /etc/httpd/sites-enabled
        fi

        avail=/etc/httpd/sites-available/$1.conf
        enabled=/etc/httpd/sites-enabled/
        site=`ls /etc/httpd/sites-available/`

        if [ "$#" != "1" ]; then
                        echo "Use script: a2ensite virtual_site"
                        echo -e "\nAvailable virtual hosts:\n$site"
                        exit 0
        else

        if test -e $avail; then
        sudo ln -s $avail $enabled
        else

        echo -e "$avail virtual host does not exist! Please create one!\n$site"
        exit 0
        fi
        if test -e $enabled/$1.conf; then

        echo "Success!! Now restart Apache server: sudo systemctl restart httpd"
        else
        echo  -e "Virtual host $avail does not exist!\nPlease see available virtual hosts:\n$site"
        exit 0
        fi
        fi

    sudo chmod +x /usr/local/bin/a2ensite

    sudo vim /usr/bin/a2dissite
        #!/bin/bash
        avail=/etc/httpd/sites-enabled/$1.conf
        enabled=/etc/httpd/sites-enabled
        site=`ls /etc/httpd/sites-enabled/`

        if [ "$#" != "1" ]; then
                        echo "Use script: a2dissite virtual_site"
                        echo -e "\nAvailable virtual hosts: \n$site"
                        exit 0
        else

        if test -e $avail; then
        sudo rm  $avail
        else
        echo -e "$avail virtual host does not exist! Exiting!"
        exit 0
        fi

        if test -e $enabled/$1.conf; then
        echo "Error!! Could not remove $avail virtual host!"
        else
        echo  -e "Success! $avail has been removed!\nPlease restart Apache: sudo systemctl restart httpd"
        exit 0
        fi
        fi

    sudo mkdir /etc/httpd/sites-available /etc/httpd/sites-enabled
    sudo vim /etc/httpd/conf/httpd.conf
       353a354
       > IncludeOptional sites-enabled/*.conf

Set up VHOST
============

Backups
=======

Create backup volume
--------------------

-  [DO console] Volumes > Add Volume > 10 GB

::

    sudo mkfs.ext4 -F /dev/disk/by-id/<volumename>
    sudo mkdir -p /mnt/backup
    sudo mount -o discard,defaults /dev/disk/by-id/<volumename> /mnt/backup
    echo /dev/disk/by-id/<volumename> /mnt/backup ext4 defaults,nofail,discard 0 0 \| sudo tee -a /etc/fstab

Set up backup
-------------

See https://www.digitalocean.com/community/tutorials/how-to-install-rsnapshot-on-ubuntu-12-04
::

    sudo yum install -y rsnapshot
    sudo yum install -y rsnapshot
    sudo vim /etc/rsnapshot.conf
        23c23
        < snapshot_root /.snapshots/
        ---
        > snapshot_root /mnt/backup/snapshots/
        40c40
        < #cmd_cp /usr/bin/cp
        ---
        > cmd_cp /usr/bin/cp
        63c63
        < #cmd_du /usr/bin/du
        ---
        > cmd_du /usr/bin/du
        67c67
        < #cmd_rsnapshot_diff /usr/local/bin/rsnapshot-diff
        ---
        > cmd_rsnapshot_diff /usr/bin/rsnapshot-diff
        93,95c93,95
        < retain alpha 6
        < retain beta 7
        < retain gamma 4
        ---
        > #retain alpha 6
        > #retain beta 7
        > #retain gamma 4
        96a97,100
        > retain hourly 6
        > retain daily 7
        > retain weekly 4
        > retain monthly 3
        120c124
        < #logfile /var/log/rsnapshot
        ---
        > logfile /var/log/rsnapshot
        229c233,234
        < #backup /var/log/rsnapshot localhost/
        ---
        > backup /var/log/rsnapshot localhost/
        > backup /var/www localhost/
    sudo rsnapshot configtest
    sudo rsnapshot -t hourly
    sudo rsnapshot hourly
    sudo vim /etc/cron.d/rsnapshot
    -  These settings will run add a snapshot to the "hourly" directory
          within our "/backup/" directory every four hours, add a daily
          snapshot everyday at 3:30 am, add a weekly snapshot every
          Monday at 3:00 am, and add a monthly snapshot on the first of
          every month at 2:30 am.
    -  It is important to stagger your backups and run larger backup
          intervals first. This means running the monthly backup first
          and progressing to shorter intervals from there in order, as
          we've done in this tutorial. This is necessary so that the
          program does not get caught up trying to do multiple backups at
          the same time, which can cause problems.
        0 \*/4 \* \* \* root /usr/bin/rsnapshot hourly
        30 3 \* \* \* root /usr/bin/rsnapshot daily
        0 3 \* \* 1 root /usr/bin/rsnapshot weekly
        30 2 1 \* \* root /usr/bin/rsnapshot monthly

Resize backup volume (only if necessary)
----------------------------------------

See https://www.digitalocean.com/community/tutorials/how-to-increase-the-size-of-a-digitalocean-block-storage-volume

-  droplet must be switched off to resize an attached volume

::

    sudo shutdown -h now

-  [DO console] Droplet loutility-server-digitalocean > Volumes > backup > More > Resize Volume > 40GB
-  [DO console] Switch On droplet
-  determine name of volume

::

    ls -l /dev/disk/by-id

    total 0
    lrwxrwxrwx 1 root root 9 Sep 21 05:47 scsi-0DO_Volume_backup -> ../../sdc
    lrwxrwxrwx 1 root root 9 Sep 21 05:44 scsi-0DO_Volume_loutility-server-backup -> ../../sdb
    lrwxrwxrwx 1 root root 9 Sep 21 05:44 scsi-0DO_Volume_loutility-server-swap -> ../../sda

-  determine filesystem type

::

    sudo lsblk --fs /dev/disk/by-id/scsi-0DO_Volume_backup

    NAME FSTYPE LABEL UUID MOUNTPOINT
    sdc ext4 0b21852e-dee8-4828-97b1-92e66d877b2d /mnt/backup

-  resize unpartitioned ext4 volume

::

    sudo resize2fs /dev/disk/by-id/scsi-0DO_Volume_backup

Set up swap volume
==================

See https://www.centos.org/docs/5/html/Deployment_Guide-en-US/s1-swap-adding.html

-  [DO console] Volumes > Add Volume > 10 GB / swapspace

::

    sudo mkswap /dev/disk/by-id/<volumename>
    sudo vim /etc/fstab # add following line
        /dev/disk/by-id/scsi-0DO_Volume_swapspace swap swap defaults 0 0
    sudo swapon -va

Security
========

Set up server level security
----------------------------

-  https://www.digitalocean.com/community/tutorials/an-introduction-to-securing-your-linux-vps

   -  https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-centos-6

   -  http://stuffphilwrites.com/2013/03/permanently-ban-repeat-offenders-fail2ban/

::

        sudo yum install -y fail2ban
        sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
        sudo vim /etc/fail2ban/jail.local
        -  set ignoreip to your personal ip address
        -  set destemail to your personal email address
        -  set enabled to true (for desired jails)
        -  set bantime to 3600 (globally)
        sudo systemctl start fail2ban
        sudo systemctl enable fail2ban

.
   -  https://www.digitalocean.com/community/tutorials/how-to-install-aide-on-a-digitalocean-vps
