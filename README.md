# Udacity Full Stack Nanodegree Project - Linux Server Configuration
Submission for [Udacity's Full Stack Web Developer Nanodegree Program](https://www.udacity.com/course/full-stack-web-developer-nanodegree--nd004)'s *Linux Server Cofiguration* Project. The project tests the developer's skill in preparing Linux servers, database server, server security vulnerabilities.

In this project, a Linux virtual machine is configured to deploy [Item Catalog App]()

Website is deployed at http://3.0.100.191.xip.io


## Configuration Summary

### 1. Create an instance in AWS Lightsail 

Go to AWS Lightsail and create a new account / sign in with your account.

Click **Create instance** and choose **Linux/Unix**, then selct **Ubuntu 16.04LTS**

Choose the cheapest plan and click **Create** button to create an instance.

### 2. Change the default SSH port from 22 to 2200
Click on **Connect using SSH** then

1. Use `sudo nano /etc/ssh/sshd_config` and then change Port 22 to Port 2200 , save & quit.
2. Reload SSH using `sudo service ssh restart`

### 3. Configure UFW

Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)

`sudo ufw default deny incoming` -- deny all incoming requests

`sudo ufw default deny outgoing`-- allow all outgoing requests

`sudo ufw allow 2200/tcp` -- allow incoming ssh request

`sudo ufw allow 80/tcp` -- allow all http request

`sudo ufw allow 123/udp` -- allow ntp request

`sudo ufw deny 22` -- deny incoming request for port 22

`sudo ufw enable` -- enable ufw

`sudo ufw status` -- check current status of ufw

### 4. Create a new user called grader and give an access 
Run `sudo adduser grader` to create a new user called `grader`

Create a new directory in sudoer directory with `sudo nano /etc/sudoers.d/grader`

Add `grader ALL=(ALL:ALL) ALL` in nano editor 

Set SSH keys for grader user with `ssh-keygen` in a local terminal. Copy the generated SSH to a virtual environment. Then allow grader user to use this key.

`su - grader`

`mkdir .ssh`

`touch .ssh/authorized_keys`

`nano .ssh/authorized_keys` and copy your generated SSH key here.

Reload SSH with `service ssh restart` Now `grader` user can login using SSH.

Disable rootlogin by  `sudo  nano /etc/ssh/sshd_config` and find `PermitRootLogin` and change it to `no`. Save exit and `service ssh restart`

### 5. Update all packages 

Run `sudo apt-get update` and `sudo apt-get upgrade`

### 6. Configure Time Zone to UTC

Run `sudo timedatectl set-timezone UTC`

### 7. Install Apache application and wsgi module

Run `sudo apt-get install apache2` to install apache 

Run `sudo apt-get install libapache2-mod-wsgi-py3` to install mod-wsgi module

Start the server `sudo service apache2 start`

### 8. Install git

Run `sudo apt-get install git`

### 9. Clone the project

Run `cd /var/www` and `sudo mkdir catalog`

Change the owner to grader `sudo chown -R grader:grader catalog`

Run `sudo chmod catalog` to give a permission to clone the project.

Switch to the `catalog` directory and clone the Catalog project.

`cd catalog` and `git clone https://github.com/maybebored/udacity-fsnd-item-catalog.git CatalogApp`

Add `catalog.wsgi` by `sudo nano catalog.wsgi` and add the following code.

```bash
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/catalog/")

from CatalogApp import app as application
application.secret_key = 'secret_key'
```

Modify filenames to deploy on AWS.

Rename `server.py` to `__init__.py` by `mv server.py  __init__.py`

### 12. Configure Apache

Create a config file `sudo nano /etc/apache2/sites-available/catalog.conf`

Paste the following code

```bash
<VirtualHost *:80>
        ServerName SERVER_IP
        ServerAdmin admin@SERVER_IP
        WSGIScriptAlias / /var/www/catalog/catalog.wsgi
        <Directory /var/www/catalog/CatalogApp/>
                Order allow,deny
                Allow from all
        </Directory>
        Alias /static /var/www/catalog/CatalogApp/static
        <Directory /var/www/catalog/CatalogApp/static/>
                Order allow,deny
                Allow from all
        </Directory>
        ErrorLog ${APACHE_LOG_DIR}/error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
Enable the new virtual host `sudo a2ensite catalog` and restart apache `sudo service apache2 restart`

### 13. Install and configure PostgressSQL

Run `sudo apt-get install postgresql postgresql-contrib`

Check if no remote connections are allowed with `sudo nano /etc/postgresql/9.3/main/pg_hba.conf`

Login to postgress `sudo su - postgres` then `psql`

Create a new user `CREATE USER catalog WITH PASSWORD 'catalog'`

Give catalog user permission to create DB: `ALTER USER catalog CREATEDB` 
and then create a new DB `CREATE DATABASE catalog WITH OWNER catalog`

Connect to the DB with `\c catalog`

Revoke all rights `REVOKE ALL ON SCHEMA public FROM public` 
`GRANT ALL ON SCHEMA public TO catalog`

Logout from postgress and return to the grader user ` \q` and `exit`

Change the engine inside Flask application.

`engine = create_engine('postgresql://catalog:catalog@localhost/catalog')`

Set up the DB with `python /var/www/catalog/CatalogApp/database_setup.py`

Populate the DB with `python /var/www/catalog/CatalogApp/populate_db.py`

Finally restart Apache with `sudo service apache2 reload` then `sudo service apache2 restart`