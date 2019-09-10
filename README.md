# Linux server configuration
Configures linux based server with AWS lightsail service to launch catalog web application

## Description
This project provides guidelines for installing and configuring linux system on AWS lightsail service for launching python based website.
You will require an AWS account to use lightsail service as prerequisite and to connect with AWS server I have used git bash for my windows system. You can use any terminal as per your suitability. you can downlod git and git bash from [here](https://gitforwindows.org/).

### Server Information
- **Server IP address:** 52.64.43.68
- **SSH port:** 2200
- **Login User:** grader
- **Application URL:** http://52.64.43.68.xip.io/catalog/

## Steps to set up the Server

### 1. Create lightsail instance
From [Amazon Lightsail](https://lightsail.aws.amazon.com/ls/webapp/home/instances) create an instance.
- Select instance location as per your location.
- Select **Linux/Unix** in Pick your instance image
- Select **OS Only** in Select a bluprint
- Select **Ubuntu 16.04 LTS** as your server operating system from many other OS options.
- In Identify your instance you can see a default name where you can change instance name if you would like so or you can keep as it is
- Finally click on **Create instance** button will create an Ubuntu instance

Have a reference from following image
![Create Instance](https://github.com/domadn1/linux-server-configuration/blob/master/CreateLightsailInstance.png)

It might take few minutes to make instance running and once it is running you can access it.

### 2. Install private key for this lightsail instance in your local machine
- First download the private key: Go to the [Account Page](https://lightsail.aws.amazon.com/ls/webapp/account/keys) and download default private key
- Downloded private key file will be something like LightsailDefaultPrivateKey-*.pem so we will rename it to lightsail_key.rsa and then move it to ~/.ssh in your local machine
- Help: if you have lightsail_key.rsa in Download folder then open git bash and you can move it with this command. `sudo mv Downloads/lightsail_key.rsa ~/.ssh/lightsail_key.rsa`
- In your terminal, type: `chmod 777 ~/.ssh/lightsail_key.rsa`

### 3. Access lightsail instance
There are many options to connect with server. Here I am using SSH. In browser where you can see running Ubuntu instance and instance description, there is also an option for connecting through browser by clicking on button **Connect using SSH**

Here I am using my own SSH client using Git bash. The Ubuntu instance provides default user **ubuntu** so first we need to connect using user ubuntu.

Now connect to the instance with user ubuntu:
`$ ssh -i ~/.ssh/lightsail_key.rsa ubuntu@52.64.43.68`

### 4. Update packages on server
Once you access server through terminal then apply following commands to install and upgrade latest packages
```console
sudo apt-get update
sudo apt-get upgrade
sudo apt-get dist-upgrade
```

### 5. Generate public key
In your local machine generate public key. Following command will generate public key which we will use for new user Grader. Give file name as lightsail_grader when it ask for file name.
```bash
$ ssh-keygen -t rsa
```

### 6. Create new User
On remote server create new user and copy lightsail_grader.pub file on server
open lightsail_grader.pub file on local machine and copy content. when you create new authorized_keys file on server then paste this content there.
```bash
$ sudo adduser grader
$ cd /home/grader/
$ sudo mkdir .ssh
$ sudo nano .ssh/authorized_keys
```

### 7. Set permissions and group and owner on .ssh
```bash
$ cd
$ sudo chown -R grader:grader .ssh
$ sudo chmod -R 700 .ssh
```
Then login as root user
```bash
$ sudo -i
```

To give access to the grader user create sudoers file as grader and paste following content to it

grader ALL=(ALL) NOPASSWD:ALL

```bash
# nano /etc/sudoers.d/grader
```

Change grader file permission and then exit from root login
```bash
# chmod 440 /etc/sudoers.d/grader
# exit
```

Open file sshd_config and modify to change ssh port and to turn off SSH passwords
```bash
$ sudo nano /etc/ssh/sshd_config
```
Port 2200 // change port to 2200

PermitRootLogin no // disable the root login

Once we complete sshd_config modification and save it safely then restart ssh using following command
```bash
$ sudo service ssh restart
```

### 8. Configure Firewall
Change default incoming policy to 'deny'
```bash
$ sudo ufw default deny incoming
```

Change default outgoing policy to 'allow'
```bash
$ sudo ufw default allow outgoing
```

Allow www, ntp and 2200/tcp
```bash
$ sudo ufw allow www
$ sudo ufw allow ntp
$ sudo ufw allow 2200/tcp
$ sudo ufw deny 5000
```

Deny port 22
```bash
$ sudo ufw deny 22
```

Enable Firewall according to this configuration
```bash
$ sudo ufw enable
```

In the Networking tab of your lightsail instance change your firewall configuration as given below 
![Network configuration](https://github.com/domadn1/linux-server-configuration/blob/master/Ubuntu-NetworkingLightsail.png)

In new terminal login with the grader user...
```bash
$ ssh -i ~/.ssh/lightsail_grader grader@52.64.43.68 -p 2200
```

### 9. Configure Timezone
Run following command and select your timezone
```bash
$ sudo dpkg-reconfigure tzdata
```

### 10. Apache2 and WSGI
Install the apache2 web server and Python3 WSGI mod
```bash
sudo apt-get install apache2 libapache2-mod-wsgi-py3
```

Disable default virtual servers:
```bash
sudo a2dissite *
```

### 11. Clone Catalog flask web application
clone [project-catalog repo](https://github.com/domadn1/project-catalog) as catalog directory
```bash
$ cd /var/www
$ sudo mkdir catalog
$ cd catalog
$ sudo git clone https://github.com/domadn1/project-catalog.git catalog
$ cd catalog
$ sudo mv app.py __init__.py
```
As per the poject-catalog instructions you need to add domain, javascript origin and redirect URIs and then download client_secret.json and copy that content
I did following configuration to google account
```code block
Domain as xip.io

Authorized JavaScript origins as http://52.64.43.68.xip.io

Authorized redirect URIs as http://52.64.43.68.xip.io/oauth2callback and http://52.64.43.68.xip.io/catalog

Create client_secret.json on server and paste that content
```

```bash
$ sudo nano /var/www/catalog/catalog/client_secret.json
```

Open database_setup.py file and replace line no. 78 ENGINE = create_engine('sqlite:///catalog.db') with ENGINE = create_engine('postgresql://catalog:catalog@localhost/catalog')
```bash
$ sudo nano catalog/database_setup.py
```
Open insert_data.py file and replace line no. 10 ENGINE = create_engine('sqlite:///catalog.db') with ENGINE = create_engine('postgresql://catalog:catalog@localhost/catalog')
```bash
$ sudo nano catalog/insert_data.py
```

Open __init__.py file

Replace line no. 31 ENGINE = create_engine('sqlite:///catalog.db?check_same_thread=False') with ENGINE = create_engine('postgresql://catalog:catalog@localhost/catalog')

Replace line no. 430,431,432 with app.run()

Also need to replace line no. 37, 38 as following
```python
CLIENT_ID = json.loads(open('/var/www/catalog/catalog/client_secret.json', 'r').read())['web']['client_id']
```

```bash
$ sudo nano catalog/__init__.py
```

### 12. WSGI file
Create catalog.wsgi file
```bash
$ cd /var/www/catalog
$ sudo nano catalog.wsgi
```
copy and paste following content to catalog.wsgi
```python
#!/var/www/catalog/venv/bin/python3

import sys
import logging

logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/catalog/")

from __init__ import app as application
application.secret_key = "newtercesisereh"
```

### 13. Virtual environment
Type following commands to install virtual environment packages
```bash
$ cd
$ sudo chmod 777 /var/www/catalog/
$ sudo pip3 install --upgrade virtualenv
$ cd /var/www/catalog
$ python3 -m virtualenv venv
$ sudo chmod 777 venv/
$ source venv/bin/activate
$ sudo apt-get install libpq-dev
$ sudo apt-get install python3-pip
$ sudo pip3 install --upgrade Flask requests sqlalchemy httplib2 requests psycopg2 psycopg2-binary python-slugify oauth2client passlib google-cloud
$ pip3 install --upgrade google-api-python-client
$ deactivate
```

### 14. Configure PostgreSQL
Install and configure PostgreSQL, and create database
```bash
$ sudo apt-get install postgresql
```

Switch to postgres user
```bash
$ sudo su - postgres
$ psql
```

Create database
```bash
# CREATE DATABASE catalog;
```

Create a role catalog and grant all privileges on catalog database
```bash
# CREATE ROLE catalog WITH LOGIN PASSWORD 'catalog';
# GRANT ALL PRIVILEGES ON DATABASE catalog TO catalog;
# \c catalog
# ALTER TABLE product_product ALTER COLUMN code TYPE text;
```

Exit PSQL and log out of postgres user:
```bash
# \q
$ exit
```

### 15. Configure Apache2 Server
Change directory to /etc/apache2/sites-available and create a file for the virtual host
```bash
$ cd /etc/apache2/sites-available
$ sudo nano catalog.conf
```
Copy and paste following contents into catalog.conf file and change your email address in ServerAdmin detail
```code block
<VirtualHost *:80>
   ServerName 52.64.43.68
   ServerAlias 52.64.43.68.xip.io
   ServerAdmin youremail@gmail.com
   WSGIDaemonProcess catalog python-path=/var/www \
     python-home=/var/www/catalog/venv
   WSGIProcessGroup catalog
   WSGIScriptAlias / /var/www/catalog/catalog/catalog.wsgi
   <Directory /var/www/catalog/catalog/>
       Require all granted
   </Directory>
   Alias /static /var/www/catalog/catalog/static
   <Directory /var/www/catalog/catalog/static/>
       Require all granted
   </Directory>
   ErrorLog ${APACHE_LOG_DIR}/error.log
   LogLevel warn
   CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
Then following command will enable site and restart the apache server
```bash
$ sudo a2ensite catalog
$ sudo service apache2 restart
```

## setup database catalog
Run database_setup.py file which will create required tables
```bash
$ python3 /var/www/catalog/catalog/database_setup.py
```

Need to alter schema for table product_item and therefore we will access postgres
```bash
$ sudo su - postgres
$ psql
# \c catalog
# ALTER TABLE product_item ALTER COLUMN description TYPE text;
# ^z
$ exit
```

Run insert_data.py file to insert data to catalog DB and restart server
```bash
$ python3 /var/www/catalog/catalog/insert_data.py
$ sudo service apache2 restart
```

Now you can access web application through configured address as ServerAlias in catalog.conf
As per this configuration you can access catalog application on this address http://52.64.43.68.xip.io/
