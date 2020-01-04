MySQL Database Management
+++++++++++++++++++++++++++++

Install and access phpMyAdmin
-----------------------------

-  https://www.digitalocean.com/community/tutorials/how-to-install-and-secure-phpmyadmin-with-apache-on-a-centos-7-server

-  sudo yum install epel-release

-  sudo yum install phpmyadmin

-  allow access from home IP address

   -  sudo vim /etc/httpd/conf.d/phpMyAdmin.conf

      -  allow/require [home workstation IP addr]

-  from browser go into phpMyAdmin (<serverIP>/phpMyAdmin)

-  log into root account, using mySQL root password

Create MySQL Database
---------------------

use phpMyAdmin to create database

-  click New

-  Under Create database

   -  set database name for <project-database> and
         <project-database-sandbox>

   -  pull down utf8_general_ci

Create MySQL Database User
--------------------------

use phpMyAdmin to create user after creating database

-  click Users

-  click Add user

   -  User name: <database-username>

   -  Host: 127.0.0.1

   -  Password: <unique password>

-  click Go

Give MySQL User Access to MySQL Database
----------------------------------------

user phpMyAdmin to give database user access to database

-  click Users

-  next to <database-username>, click Edit Privileges

-  Click Database

   -  Add privileges on the following databases

   -  select <project-database> and <project-database-sandbox>

   -  click Go

   -  check Check All

   -  click go

database dump / restore
-----------------------

-  mysqldump -h [server] -u [user] -p[password] db1 > file.sql

-  mysql -h localhost -u scoretility -p scoretility < file.sql
