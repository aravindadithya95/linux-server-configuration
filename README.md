# Linux Server Configuration
In this project, I take a baseline installation of a Linux server on a virtual machine and prepare it to host web applications by securing the server from a number of attack vectors, install and configure a database server, and deploy an existing web application onto it.

This project is part of the [Full Stack Web Developer Nanodegree](https://www.udacity.com/course/full-stack-web-developer-nanodegree--nd004) at _Udacity_.

## Server
- URI: `http://ec2-34-226-216-216.compute-1.amazonaws.com`
- Public IP: `34.226.216.216`
- SSH port: `2200`

## Configuration
### Update packages
- Update available packages list by running `sudo apt-get update`.
- Upgrade installed packages by running the following commands:
```
sudo apt-get upgrade
sudo apt-get dist-upgrade
```

### Change the SSH port
- Open the ssh config file using `sudo vi /etc/ssh/sshd_config`.
- Change the line that says `Port 22` to `Port 2200`.
- Save and exit the file.
- Restart the `sshd` service by running `sudo service sshd restart`.
- Add a rule to accept TCP connections on port 2200 in the AWS console.

### Configure the firewall
- Make sure the firewall is inactive using `sudo ufw status` before we configure the ports.
- Allow all outgoing connections using `sudo ufw default allow outgoing`.
- Deny all outgoing connections initially using `sudo ufw default deny incoming`, so that we can configure which connections to allow.
- Allow ssh connections using `sudo ufw allow ssh`.
- Allow incoming tcp requests at port 2200 using `sudo ufw allow 2200/tcp`.
- Allow incoming http requests at port 80 using `sudo ufw allow www`
- Allow NTP using `sudo ufw allow ntp`.
- You can check the rules you just added using `sudo ufw show added`.
- Enable the firewall using `sudo ufw enable`.
- You can confirm using `sudo ufw status`.

### Create and setup the user **grader**
- Create the user using `sudo adduser grader` and enter any additional information if required.
- To give **grader** `sudo` access:
  - Run `sudo vi /etc/sudoers.d/grader`.
  - Add the following line to the file: `grader ALL=(ALL) NOPASSWD:ALL`.
  - Save and exit the file.
- To create an RSA key pair, run `ssh-keygen` on a new terminal window or tab in your local machine and complete the setup, after which two files will be generated in the name specified, one with a _.pub_ extension.
- In the VM, run `sudo su - grader` to login as grader and then run the following commands to setup an `authorized_keys` file for RSA encryption:
```
mkdir .ssh
vi .ssh/authorized_keys
```
- Copy the contents of the file with the _.pub_ extension that was created in your local machine and save it in the `authorized_keys` file from the last step.
- Secure the _.ssh_ directory and the `authorized_keys` file, by setting file permissions using:
```
chmod 700 .ssh
chmod 644 .ssh/authorized_keys
```
- The user **grader** can now login to the VM using the file that was generated along with the _.pub_ file like:
```
ssh grader@13.58.217.237 -i <file_name> -p 2200
```
- To force key-based authentication(default in AWS), open the ssh config file that we worked on earlier:
```
sudo vi /etc/ssh/sshd_config
```
- Change the line that reads `PasswordAuthentication yes` to `PasswordAuthentication no`.
- To prohibit `root` login, change the line that reads `PermitRootLogin prohibit-password` to  `PermitRootLogin no`
- Restart the `sshd` service by running `sudo service ssh restart`

### Configure the local timezone
- Configure the local timezone to UTC
```
sudo timedatectl set-timezone UTC
```
- You can check using the `date` command.

### Deploy the Item Catalog project
#### Install Apache and the mod-wsgi application handler
- Install Apache
```
sudo apt-get install apache2
```
- Install the **mod-wsgi** application handler
```
sudo apt-get install libapache2-mod-wsgi
```
- To enable mod_wsgi, run the following command:
```
sudo a2enmod wsgi
```

#### Install and setup PostgreSQL
- Install **PostgreSQL**
```
sudo apt-get install postgresql postgresql-contrib
```
- Create the user **catalog** in PostgreSQL and **catalog** as the password at the prompt:
```
sudo -u postgres createuser catalog -P
```
- Create the database **catalog**:
```
sudo -u postgres createdb catalog
```

#### Install git, python-pip and virtualenv
```
sudo apt-get install git
sudo apt-get install python-pip
sudo pip install virtualenv
```

#### Directory Structure
- To setup the directory structure for the Flask application, first:
```
cd /var/www
```
- Create the directory _catalog_ and `cd` into it
```
sudo mkdir catalog
cd catalog
```
- Clone the _Item Catalog_ repository
```
sudo git clone https://github.com/aravindadithya95/item-catalog.git
```
-  Rename the directory to **catalog** and `cd` into it
```
sudo mv item-catalog/ catalog/
cd catalog
```
-  Rename the `views.py` file to `__init__.py`:
```
sudo mv views.py __init__.py
```
- Change the line `engine = create_engine('postgresql:///catalog')` to `engine = create_engine('postgresql://catalog:catalog@localhost/catalog')` in `__init__.py`, `models.py` and `populate_db.py`.
- Change file I/O operations' argument from relative to absolute paths in `__init__.py`.

#### Install Flask and other dependencies
- We want to install Flask and other dependencies in a virtual environment to isolate them from the main operating system.
- Give the following command (where venv is the name you would like to give your temporary environment):
```
sudo virtualenv venv
```
- Activate the virtual environment with the following command:
```
source venv/bin/activate
```
- Install Flask and other dependencies using pip with:
```
sudo pip install psycopg2
sudo pip install flask
sudo pip install sqlalchemy
sudo pip install oauth2client
sudo pip install requests
sudo pip install flask_httpauth
sudo pip install itsdangerous
```
- To deactivate the environment, give the following command:
```
deactivate
```

#### Configure and Enable a New Virtual Host
- Create the `catalog.conf` file
```
sudo vi /etc/apache2/sites-available/catalog.conf
```
- Add the following lines of code to the file to configure the virtual host
```
<VirtualHost *:80>
		ServerName 34.226.216.216
    ServerAlias ec2-34-226-216-216.compute-1.amazonaws.com
		ServerAdmin admin@34.226.216.216
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
- Enable the virtual host with the following command
```
sudo a2ensite catalog
```

#### Create the .wsgi file
- Create the `catalog.wsgi` file
```
sudo vi /var/www/catalog/catalog.wsgi
```
- Add the following lines of code
```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/catalog/")

from catalog import app as application
application.secret_key = 'catalog_secret_key'
```

- The final directory structure should look like this:

|--------catalog  
|----------------catalog  
|-----------------------static  
|-----------------------templates  
|-----------------------venv  
|-----------------------__init__.py  
|----------------catalog.wsgi

### Restart Apache
```
sudo service apache2 restart
```

## References
- [Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
