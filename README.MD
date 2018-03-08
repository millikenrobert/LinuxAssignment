# Linux Setup
## Overview
This document describes the processes that are required to setup the Udacity Catalog application on AWS Lightsail.

## Version
1.0 Initial submission for review
1.1 Had to run `sudo apt-get dist-upgrade` to update

## Compatibility
The Linux server is setup with the required applications and packages.

## Usage Notes
Website can be access at http://ec2-13-211-172-94.ap-southeast-2.compute.amazonaws.com/

The reviewer has been supplied with the local key. Connnection to the Server should be via

*ssh -i ~/.ssh/{key.pem} grader@13.211.172.94 -p 2200*

Where {key.pem} is the key file.
SSH is via port 2200
IP is 13.211.172.94



# Step-by-step Setup

## Set the key for SSH access so that the server can be accessed locally
1. Copy key to .ssh
* `mv key.pem  ~/.ssh/keyPair.pem`
2. Change the permissions
* `chmod 600 ~/.ssh/keyPair.pem`
3. Confirm the server can be accessed by
* `ssh -i ~/.ssh/keyPair.pem ubuntu@54.206.12.65`

## Create the Grader user
1. Add user (pw:graderpw)
* `sudo adduser grader`
2. Add grader file
* `sudo cp /etc/sudoers.d/README /etc/sudoers.d/grader`
3. Edit grader file
* `sudo nano /etc/sudoers.d/grader`
4. Add the following
* `grader ALL=(ALL:ALL) ALL NOPASSWD:ALL`
5. Confirm grader has sudo permissions
* `sudo su - grader`
* `sudo ls -al`

## Update the packages
1. Find the available package updates
* `sudo apt-get update`
2. Upgrade the packages
* `sudo apt-get upgrade`
* `sudo apt-get dist-upgrade`

## Update the firewall
1. Allow an exception in Lightsail for ssh 2200, deny 22 and add 123 (UDP)
2. Change SSH port to 2200
* `sudo nano /etc/ssh/sshd_config`
3. Restart Service
* `sudo service ssh restart`
4. Check ufw is off
* `sudo ufw status`
5. Deny all incoming, Allow outgoing
* `sudo ufw default deny incoming`
* `sudo ufw default allow outgoing`
6. Allow ssh, http and ntp
* `sudo ufw allow ssh`
* `sudo ufw allow 2200/tcp`
* `sudo ufw allow www`
* `sudo ufw allow 123/udp`
* `sudo ufw deny 22`
7. Enable Firewall
* `sudo ufw enable`
8. Confirm SSH does not work on Port 22, but does on 2200
* `ssh -i ~/.ssh/keyPair.pem ubuntu@13.211.172.94 -p 2200`

## Update the TZ to UTC
1. Change "none of the above" the "UTC"
* `sudo dpkg-reconfigure tzdata`

## Install Apache
1. Install the Apache package:
* `sudo apt-get install apache2`
2. Check that http://13.211.172.94/ returns the Apache Website
True!

## Install WSGI
1. Install the libapache2-mod-wsgi package
* `sudo apt-get install libapache2-mod-wsgi python-dev`
2. Enable wsgi
* `sudo a2enmod wsgi`
3. Start Apache
* `sudo service apache2 start`


## Install git
1. Install the git package
* `sudo apt-get install git`
2. Change the owner
* `cd /var/www`
3. Make catalog
* `sudo mkdir catalog`
4. Change the catalog folder permissions
* `sudo chown -R grader:grader catalog`
5. Enter the catalog folder
* `cd catalog`


## Clone and modify my catalog project
1. Create the directory to clone to
* `sudo git clone https://github.com/millikenrobert/catalog.git catalog`
2. Create catalog.wsgi
* `nano catalog.wsgi`

```
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
application.secret_key = 'supersecretkey'
```

3. Rename application.py to __init__.py
* `mv application.py __init__.py`


## Install Virtual Environment
1. Install pip
* `sudo apt-get install python-pip`
2. Install VirtualEnv
* `sudo pip install virtualenv`
3. Create the virtual environment
* `sudo virtualenv venv`
4. Launch the virtual environment
* `source venv/bin/activate`
5. Set permissions
* `sudo chmod -R 777 venv`

## Install Packages
1. Install all below packages
* `pip install Flask`
* `pip install sqlalchemy`
* `pip install oauth2client`
* `pip install httplib2`
* `pip install psycopg2`
* `pip install requests`


## Add OAuth credentials in Google
1. Add new OAuth Credentials
Authorized JavaScript: http://ec2-13-211-172-94.ap-southeast-2.compute.amazonaws.com
Authorized JavaScript: http://13.211.172.94
Authorized redirect URIs: http://ec2-13-211-172-94.ap-southeast-2.compute.amazonaws.com/oauth2callback
2. Update the client_secrets.json file with new details
3. Update the templates/login.html file to new ClientID
980237270317-5uvlf6g5b7erdcf8o2fldcvup2nf3jle.apps.googleusercontent.com



## Create the config file for the virtual server
1. Go to the config directory
* `sudo nano /etc/apache2/sites-available/catalog.conf`

And enter

```

<VirtualHost *:80>
    ServerName 13.211.172.94
    ServerAlias http://ec2-13-211-172-94.ap-southeast-2.compute.amazonaws.com/
    ServerAdmin admin@13-211-172-94
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



```

2. Enable virtual host

## Setup for PostreSQL
1. Install Python PostgreSQL dependencies
* `sudo apt-get install libpq-dev python-dev`
* `sudo apt-get install postgresql postgresql-contrib`


## Add the PostgresSQL user named catalog
1. Switch to the postgres user
* `sudo su - postgres`
2. Connect to the psql server
* `psql`
3. Create the user
* `CREATE ROLE catalog WITH LOGIN;`
4. Set catalog role to create databases
* `ALTER ROLE catalog CREATEDB;`
5. Set the catalog PW
* `\password catalog`
Pw: catalogpw123
6. Exit psql
* `\q`


## Create the database
1. Create the catalog database as the Linux catalog user
* `CREATE DATABASE catalog WITH OWNER catalog;`
* `\c catalog`
* `REVOKE ALL ON SCHEMA public FROM public;`
* `GRANT ALL ON SCHEMA public TO catalog;`
* `\q`


## Update the DB references
1. Update PostgreSQL engine in "__init__.py" and "database_setup.py" and populateDb.py
* engine = create_engine('postgresql://catalog:catalogPW617@localhost/catalog')


## Reset the apache server
1. Restart the apache server
* `sudo service apache2 restart`

# A review of the Rubrik - Sanity Check
## User Management
* Can you log into the server as the user grader using the submitted key? Yes, I have tested
* Is remote login of the root user disabled? Yes, no root comes with Lightsail. Only the Ubuntu user and the Grader user exist.
* Is the grader user given sudo access? Yes, grader has been added as a sudoer

## Security
* Is the firewall configured to only allow for SSH, HTTP, and NTP? Yes, both AWS and UFW have blocked all connections exceppt for 2200, 80 and 123. 22 has been denied.
* Are users required to authenticate using RSA keys? Yes, SSH is enforced.
* Are the applications up-to-date? Yes, Update and Upgrade has been completed
* Is SSH hosted on non-default port? Yes, hosted on 2200.

## Application Functionality
* Is there a web server running on port 80? Yes, the url resonds to both the alias and ip.
* Has the database server been configured to properly serve data? Yes, the website works.
* Has the web server been configured to serve the Item Catalog application? Yes, catalog site is served.

## Documentation
Is a README file included in the GitHub repo containing all specified information? Yes!


# References and third-party resources
* Udacity course (www.udacity.com)
* Amazon for getting keyGen pair done(https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html#ec2-launch-instance)
* Ask Ubuntu where i locked myself out of sudo (https://askubuntu.com/)
* Some areas i wasnt sure of the following helped (https://github.com/iliketomatoes)
