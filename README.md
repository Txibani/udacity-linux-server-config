# Linux Server Configuration

## About

Linux server configuration project from Udacity Full Stack Nanodegree Lesson 5, Deploying to Linux servers. 
Baseline installation of a Linux server and prepared it to host the web application Item Catalog from lesson 4. Secured server from a number of attack vectors, installed and configured a database server, and deployed the web application into it.

URL to see the deployed web application: http://3.8.115.252.xip.io/

## Project steps

### Create a server
1. Create an Amazon Lightsail instance
2. Add port Custom TCP 2200 (inside lightsails)
3. Download lightsail key and move it to .ssh/
4. `chmod 700 ~/.ssh/`
5. `chmod 600 ~/.ssh/LightsailDefaultPrivateKey.pem`
6. `ssh ubuntu@3.8.115.252 -i ~/.ssh/LightsailDefaultPrivateKey.pem`


### Update/upgrade packages
All system packages have been updated to most recent versions:
1. `sudo apt-get update`
2. `sudo apt-get upgrade`
3. `sudo apt-get autoremove`


### Secure server
Configured UFW (Uncomplicated Firewall):
1. Check status with `sudo ufw status`
2. Block all incoming requests `sudo ufw default deny incoming`
3. Allow all outgoing requests `sudo ufw default allow outgoing` 
4. Allow SSH on port 2200 `sudo ufw allow 2200/tcp`
5. Allow HTTP on port 80 `sudo ufw allow www`
6. Allow NTP on port 123 `sudo ufw allow ntp`
7. Check added ports to UFW `sudo ufw show added`
8. Turn on the firewall by typing `sudo ufw enable` 
9. Check status with `sudo ufw status`
10. `sudo nano /etc/ssh/sshd_config` to change SSH port from 22 to 2200 (line 5 updated from 22 to 2200)
11. Restart ssh service by running `sudo service ssh restart`
11. Finally connect to port 2200 `ssh ubuntu@3.8.115.252 -p 2200 -i ~/.ssh/LightsailDefaultPrivateKey.pem` 


### Createa new user grader and give access
1. Create new user grader `sudo adduser grader`
2. Give sudo access:
    2.1 Edit grader sudoer file `sudo nano /etc/sudoers.d/grader`
    2.2 Add to the file `grader ALL=(ALL:ALL) ALL` and save
3. To switch to grader user run `su - grader` and enter user grader password
4. Generate a ssh key on the local machine for grader `ssh-keygen` and name it graderKey
5. Copy the generated public key `graderKey.pub`
6. Place generated key on the remote server `graderKey.pub` for that:
    6.1 Log in into the server as grader
    6.2 Create .ssh folder if there is none `sudo mkdir .ssh` within home directory
    6.3 Create file authorized_keys to store all public keys `sudo touch .ssh/authorized_keys` 
    6.4 Copy content from graderkey.pub and past it on the authorize_keys file created before `sudo nano .ssh/authorized_keys` and save it
    6.5 Make sure shh folder belongs to grader user `sudo chown -R grader:grader .ssh`
    6.5 Set file permissions `sudo chmod 700 .ssh` & `sudo chmod 644 .ssh/authorized_keys`
    6.6 Test key pair
    6.7 Make sure key-based authentication is forced `sudo nano /etc/ssh/sshd_config` - PasswordAuthentication to no
    6.8 Restart ssh service `sudo service sshd reload`
    6.9 SSH into the server as grader `ssh grader@3.8.115.252 -p 2200 -i ~/.ssh/graderKey`

### Prepare to deploy the project
1. Configure the local timezone to UTC `sudo timedatectl set-timezone UTC`
2. Install Apache and mod_wsgi for python3 for that run:
    2.1 To install Apache `sudo apt-get install apache2`
    2.1 To install mod_wsgi `sudo apt-get install libapache2-mod-wsgi python-dev`
3. Install and configure PostgreSQL
    3.1 Install postgresql `sudo apt-get install postgresql`
    3.2 Create new database user named catalog `sudo -u postgres createuser -P catalog`
    3.3 Login as postgres User (Default User), and get into PostgreSQL shell:
        3.3.1 `sudo su - postgres`
        3.3.2 `psql`
        3.3.3 Create a database named catalog `CREATE DATABASE catalog;`
        3.3.4 Create user catalog `CREATE USER catalog;`
        3.3.5 Change password for user catalog `ALTER ROLE catalog WITH PASSWORD 'catalog';` 
        3.3.6 Give user "catalog" permission to "catalog" application database `GRANT ALL PRIVILEGES ON DATABASE catalog TO catalog;`
        3.3.7 Quit postgreSQL `\q` and exit from user postgress `exit`


### Clone and setup the Item Catalog project from the Github repository
1. Install git `sudo apt-get install git`
2. Create FlaskApp folder `sudo mkdir /var/www/FlaskApp`
3. Move to FlaskApp folder `cd /var/www/FlaskApp`
4. Give grader only access `sudo chown -R grader:grader /var/www/FlaskApp`
5. Clone repo with item catalog project from github `sudo git clone git@github.com:Txibani/item-catalogue.git`


### Update item catalog project as follows:
1. Rename project.py to __init__.py by running `sudo mv project.py __init__.py`
2. Find any reference to client_secret.json and replace it with its full path name `/var/www/FlaskApp/FlaskApp/client_secret.json`
3. Find line `engine = create_engine('sqlite:///itemcataloguewithusers.db?check_same_thread=False')` and replace it with `engine = create_engine('postgresql://catalog:DB-PASSWORD@localhost/catalog')`


### Setup for deploying a Flask App on Ubuntu 
1. Install pip `sudo apt-get install python-pip`
3. Install virtualenv `sudo pip install virtualenv`
    3.1 venv is the name of the temporary environmen `sudo virtualenv venv`
    3.2 Install Flask in that environment by activating the virtual environment `source venv/bin/activate` then `sudo pip install Flask`
    3.3 Install packages needed by project through pip in venv:
        3.2.1 `sudo pip install httplib2`
        3.2.2 `sudo pip install requests`
        3.2.3 `sudo pip install --upgrade oauth2client`
        3.2.4 `sudo pip install Flask`
        3.2.5 `sudo apt-get install libpq-dev`
        3.2.6 `sudo pip install psycopg2`
        3.2.7 `pip install flask-sqlalchemy`
    3.4 Run `sudo python database_setup.py`
    3.4 Run `sudo python lotsofcategories.py`
    3.4 Run `sudo python __init__.py` It should display “Running on http://localhost:5000/” or "Running on http://127.0.0.1:5000/". If you see this message, you have successfully configured the app.
    3.5 Run `deactivate`

### Configure and Enable a New Virtual Host
1. Run `sudo nano /etc/apache2/sites-available/FlaskApp.conf`
2. Add
```
<VirtualHost *:80>
    ServerName 3.8.115.252
    ServerAdmin admin@3.8.115.252
    WSGIScriptAlias / /var/www/FlaskApp/FlaskApp.wsgi
    <Directory /var/www/FlaskApp/FlaskApp/>
        Order allow,deny
        Allow from all
        Options -Indexes
    </Directory>
    Alias /static /var/www/FlaskApp/FlaskApp/static
    <Directory /var/www/FlaskApp/FlaskApp/static/>
        Order allow,deny
        Allow from all
        Options -Indexes
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
3. Disable the default virtual host `sudo a2dissite 000-default.conf`
4. Enable the virtual host with the following command: `sudo a2ensite FlaskApp`

### Create and configure the .wsgi File
1. Move to FlaskApp folder cd /var/www/FlaskApp
2. Create file `sudo touch /var/www/FlaskApp/FlaskApp.wsgi`
3. Add content below to this file and save:
```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/FlaskApp/")

from FlaskApp import app as application
application.secret_key = 'super secret key'
```
4. Finally restart apache `sudo service apache2 restart`


### Check deployed project
1. Go to http://3.8.115.252.xip.io/
2. If server error occurs check logs by running `sudo tail -f /var/log/apache2/error.log`


### RESOURCES
- https://medium.com/@mariasurmenok/creating-a-server-with-amazon-lightsail-11c377cf814c
- https://askubuntu.com/questions/138423/how-do-i-change-my-timezone-to-utc-gmt
- https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
- https://www.calhoun.io/how-to-install-postgresql-9-5-on-ubuntu-16-04/
