Linux User Account Creation
+++++++++++++++++++++++++++++++

-  https://www.digitalocean.com/community/tutorials/initial-server-setup-with-centos-7

.. code-block:: shell

    sudo adduser <username>
    sudo passwd <username>

Following only to give sudo privilege - NOT FOR APACHE ACCOUNT

.. code-block:: shell

    sudo gpasswd -a <username> wheel


Only do the following if setting up account used by another
administrator

.. code-block:: shell

   chage --lastday 0 <username> # requires password change, see https://www.tecmint.com/force-user-to-change-password-next-login-in-linux/
   chage -l <username> # verify password expiration

From <username> account, enable history time display by adding the following lines to `.bashrc`

.. code-block:: shell

    # User specific aliases and functions
    export HISTTIMEFORMAT="%Y-%m-%d %H:%M "

.. note::
    need to log out / log in for this to take affect

Only do the following if setting up account controlled by this
administrator

.. code-block:: shell

    mkdir .ssh
    chmod 700 .ssh
    [generate ssh keys on local development platform using PuTTYgen]
    vi .ssh/authorized_keys [insert public ssh key just generated]
    chmod 600 .ssh/authorized_keys
    exit

Set up PuTTY to use private key

-  Open PuTTY
-  Enter <host IP address> as Host Name (or IP address)
-  Make sure Connection type is SSH
-  On the left, click Connection > SSH > Auth
-  Under Private key file for authentication, browse to private key file (.ppk) generated using PuTTYgen
-  Click back to Session on the left
-  Enter a name for a Saved Session (e.g., loutility-server-digitalocean-<username>)
-  Click Save (now you can restore this stuff without typing)
-  Click Open to verify the session works
-  The window should log in without you needing to enter a password

Add to digitalocean team
========================

From digitalocean.com (sign in)

-  ACCOUNT > Team > Invite Members

-  enter email address, cliick Invite Members
