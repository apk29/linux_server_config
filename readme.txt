Project Overview

Prepare a baseline installation of a Linux server and prepare it to host a web application. Secure the server from a number 
of attack vectors, install and configure a database server with a web application. 

Server Information

IP address: 52.25.36.126

Accessible SSH port: 2200

Application URL: http://ec2-52-25-36-126.us-west-2.compute.amazonaws.com

Configuration Server

From the home page in your Amazon Lightsail home page, click on the lightsail terminal icon on the top right corner of your server.

      1. Create a new user grader in the lightsail terminal and give sudo status to the user
         -sudo adduser grader
       
          The lightsail server does not allow for password authorization, you need to 
          go to you ssh_config and allow this first; so you get login remotely.
          The lightsail terminal is not easy to work with; you can’t copy and paste easily.
            -Edit the ssh_config using sudo nano /etc/ssh_config, 
            -StrictModes no
            -update PasswordAuthentication to yes
            -control o, enter, control x, use these command after editing files with nano
            -sudo service ssh restart

        Login in from your own terminal git bash with the grader using the following command:
            -ssh grader@publicIp –p 22, example: ssh grader@34.207.135.9 –p 2
       
        Give sudo authority to your grader user
           -sudo nano /etc/sudoers.d/grader
           -paste the following:
            grader ALL=(ALL) NOPASSWD:ALL
           -To verify sudo access, sudo cat /etc/passwd
           -after finishing up your setup, you should come back and change this file to:
            grader ALL=(ALL) PASSWD:ALL, the password will be the admin password that you setup

      2.  Generate Key Pairs     
           -Open a new terminal on your local drive.
           -ssh-keygen
           -save the key to /c/Users/computername/.ssh/id_rsa
               Example: /c/Users/vostro420/.ssh/linuxserver
           -you do not need a password for the key, you’ll be using the keys you just 
            to automatically login, if you added one, you can remove it by using:
            ssh-keygen –p –f  ~/.ssh/id_dsa 

       3. Update your SSh port from 22 to 2200
           -go back to the terminal you were logged into 
           - sudo nano /etc/ssh/sshd_config
           - at about the fifth line down, Port 22 to Port 2200
           - logout and log back in, using 
             ssh userName@public ip -p 22 -i ~/.ssh/keyName
             example: ssh grader@34.207.135.9 –p 22 –i~.ssh/linuxserver

       4. Adding the key pair back to the server
          - go to the location of the key you just generated in your local drive, 
            open the one key file ending in .pub and select all, copy 
          - you should still be logged in from updating the port number,
          - mkdir .ssh
          - touch .ssh/authorized_key
          - nano .ssh/authorized_key, paste the key you just copied 

       5.  DO NOT LOG OUT UNTIL YOU HAVE BEEN ABLE TO LOGIN FROM 
             ANOTHER TERMINAL

       6. Authorize file permission to the authorized key file
          -be in the grader server, 
          -chmod 700 .ssh
          -chmod 644 .ssh/authorized_key
          -Logout and login: ssh userName@public ip -p 22 -i ~/.ssh/keyName
          -sudo nano /etc/ssh/sshd_config, edit PasswordAuthentication back to no
           After this step, you should not need to type in your password to login
          -sudo service ssh restart

7. Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)
            -sudo ufw default deny incoming
             -sudo ufw default allow outgoing
             -sudo ufw allow ssh
             -sudo ufw allow 2200
             -sudo ufw allow www
             -sudo ufw allow 123
             -sudo ufw deny 22
             - sudo nano /etc/ssh/ssh_config, change the 22 to 2200 
             -sudo ufw enable, y
             -sudo ufw status
	     -Go to lightsail and update your port there as well.
             -from your server icon, on the top right corner, click on the three vertical
              dots, choose manage, choose the networking tab:
               it should look like this:
              Application      Protocol     Port range
              HTTP               TCP           80
              Custom             UDP          123
              Custom             TCP           2200



8. Configure the local timezone to UTC
   - sudo dpkg-reconfigure tzdata
9. Install Apache
    -sudo apt-get install apache2

10. Install mod_wsgi
     - sudo apt-get install libapache2-mod-wsgi python-dev
     - sudo a2enmod wsgi
     - sudo service apache2 start

11. Clone your Catalog app from GitHub
    -sudo sudo apt-get install git
    -cd /var/www
    -sudo mkdir catalog
    -sudo chown –R grader:grader catalog
    -cd /catalog
    -git clone https://github.com/catalog_location_in_your_git
    -create a catalog.wsgi file
     Copy the following into your catalog.wsgi file:
Import sys
Import logging
Logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, “/var/www/catalog/”)

from catalog import app as application
application.secret_key = ‘super_secret_key’ (match this to your secret key)

   -rename your application.py file to __init__.py, this is your main python file
    Mv application.py __init__.py

12. Install the virtual environment for python called virtualenv
   -sudo pip install virtualenv
   -sudo virtualenv venv
   -source venv/bin/activate
   -sudo chmod –R 777 venv
   -you don’t need to exit, but the command to exit is 
     Deactivate

13. Install Flask and dependencies
   - pip install Flask
   - sudo pip install httplib2 oauth2client sqlalchemy psycopg2 sqlalchemy_utils

14. Update Client_secrets.json in the __init__.py, database_setup.py
   -nano __init__.py
   -you need to update the client_secret.json to make it look like 
     This: /var/www/catalog/catalog/client_secrets.json
   -There are five locations, first one is located near the top, the last four are located 
     under the def fbconnect(): & def gconnect(): sections
         

15. Configure and enable the virtual host
      -sudo nano /etc/apache2/sites-available/catalog.conf
      -paste this:

<VirtualHost *:80>
    ServerName 52.25.36.126
    ServerAlias ec2-52-25-36-126.us-west-2.compute.amazonaws.com
    ServerAdmin admin@52.25.36.126
    WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/venv/lib/python2.7/site-packages
    WSGIProcessGroup catalog
    WSGIScriptAlias / /var/www/catalog/catalog.wsgi
    <Directory /var/www/catalog/catalog/>
        Order allow,deny
        Allow from all
    </Directory>
    Alias /static /var/www/catalog/catalog/static
    <Directory /var/www/catalog/catalog/static/>
        Order allow,deny
        Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

           -sudo a2ensite catalog

    16. Install and configure PostgresQL
          -sudo apt-get install libpq-dev python-dev
          -sudo apt-get install postgresql postgresql-contrib
          -sudo –i –u postgres
          -psql
          -create user catalog with password ‘password’;
          -alter user catalog created;
          -create database catalog with owner catalog;
          -\c catalog
          -revoke all on schema public from public;
          -grant all on schema public to catalog;
          -\q
          -exit
   17. Update the __init__.py & database_setup.py files
         -Update engine=create_engine(‘postgresql://catalog:password@localhost/catalog’)

   18. Make sure the pg_hba.conf reflects the following:
         Local     postgres user          postgres                                    peer
         Local     all                            all                                 peer
         Local     all                            all               127.0.0.1/32      md5
         Local     all                            all                ::1/128          md5         

   19. sudo service apache2 restart

   20: Go to http://52.25.36.126

   21: Optional Automatic Updates
       https://help.ubuntu.com/community/AutomaticSecurityUpdates
       -sudo apt-get install unattended-upgrades
       -to see current configuration, /etc/apt/apt.conf.d/50unattended-upgrades
       -to make updates, /etc/apt/apt.conf.d/10periodic, this will update the package list and security updates
        daily and clean the local download archive weekly
       -review the logs, /var/log/apt/unattended-upgrades/unattended-upgrades.log

        APT::Periodic::Update-Package-Lists "1";
        APT::Periodic::Download-Upgradeable-Packages "1";
        APT::Periodic::AutocleanInterval "7";
        APT::Periodic::Unattended-Upgrade "1";





          













 

