# Change MySQL Data Directory in CentOS
Brief HowTo to change MySQL data directory location. 

## Step 1 — Moving the MySQL Data Directory

To prepare for moving MySQL’s data directory, let’s verify the current location by starting an interactive MySQL session using the administrative credentials.
´´
    mysql -u root -p
´´

When prompted, supply the MySQL root password. Then from the MySQL prompt, select the data directory:
´´
    select @@datadir;

Output
+-----------------+
| @@datadir       |
+-----------------+
| /var/lib/mysql/ |
+-----------------+
1 row in set (0.00 sec)
´´

This output confirms that MySQL is configured to use the default data directory, /var/lib/mysql/, so that’s the directory we need to move. Once you've confirmed this, type exit and press “ENTER” to leave the monitor:
´´
    exit
´´

To ensure the integrity of the data, we’ll shut down MySQL before we actually make changes to the data directory:

´´
    sudo systemctl stop mysqld
´´
or 

´´
  sudo services mysql stop
´´

systemctl doesn't display the outcome of all service management commands, so if you want to be sure you've succeeded, use the following command:
´´
    sudo systemctl status mysqld
´´
or 

´´
services mysqld status
´´

You can be sure it’s shut down if the final line of the output tells you the server is stopped:

´´
Output
. . .
Jul 18 11:24:20 ubuntu-512mb-nyc1-01 systemd[1]: Stopped MySQL Community Server.
´´

Now that the server is shut down, we’ll copy the existing database directory to the new location with rsync. Using the -a flag preserves the permissions and other directory properties, while-v provides verbose output so you can follow the progress.

Note: Be sure there is no trailing slash on the directory, which may be added if you use tab completion. When there’s a trailing slash, rsync will dump the contents of the directory into the mount point instead of transferring it into a containing mysql directory:


    sudo rsync -av /var/lib/mysql /mnt/volume-nyc1-01

Once the rsync is complete, rename the current folder with a .bak extension and keep it until we’ve confirmed the move was successful. By re-naming it, we’ll avoid confusion that could arise from files in both the new and the old location:


    sudo mv /var/lib/mysql /var/lib/mysql.bak
    
Now we’re ready to turn our attention to configuration.

## Step 2 — Pointing to the New Data Location

MySQL has several ways to override configuration values. By default, the datadir is set to /var/lib/mysql in the /etc/my.cnf file. Edit this file to reflect the new data directory:

    sudo vi /etc/my.cnf

Find the line in the [mysqld] block that begins with datadir=, which is separated from the block heading with several comments. Change the path which follows to reflect the new location. In addition, since the socket was previously located in the data directory, we'll need to update it to the new location:
/etc/my.cnf

[mysqld]
. . .
datadir=/mnt/volume-nyc1-01/mysql
socket=/mnt/volume-nyc1-01/mysql/mysql.sock
. . .

After updating the existing lines, we'll need to add configuration for the mysql client. Insert the following settings at the bottom of the file so it won’t split up directives in the [mysqld] block:
/etc/my.cnf

[client]
port=3306
socket=/mnt/volume-nyc1-01/mysql/mysql.sock

When you’re done, hit ESCAPE, then type :wq! to save and exit the file.
Step 3 — Restarting MySQL

Now that we've updated the configuration to use the new location, we're ready to start MySQL and verify our work.

    sudo systemctl start mysqld
    sudo systemctl status mysqld

To make sure that the new data directory is indeed in use, start the MySQL monitor.

    mysql -u root -p

Look at the value for the data directory again:

    select @@datadir;

Output
+----------------------------+
| @@datadir                  |
+----------------------------+
| /mnt/volume-nyc1-01/mysql/ |
+----------------------------+
1 row in set (0.01 sec)

Now that you’ve restarted MySQL and confirmed that it’s using the new location, take the opportunity to ensure that your database is fully functional. Once you’ve verified the integrity of any existing data, you can remove the backup data directory with sudo rm -Rf /var/lib/mysql.bak.


Reference: https://www.digitalocean.com/community/tutorials/how-to-change-a-mysql-data-directory-to-a-new-location-on-centos-7
