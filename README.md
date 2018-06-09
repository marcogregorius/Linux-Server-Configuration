# Linux Server Configuration

This README shows how to set up this [Item Catalog Website](https://github.com/marcogregorius/item_catalog)
on a Ubuntu Server hosted with Amazon Lightsail service. Includes installing updates, securing the
server from a number of attack vectors, and installing/configuring web and database servers.

## Server Information
- IP Address: 54.255.161.210
- SSH port: 2200
- SSH connection command as `grader`:

`ssh grader@54.255.161.210 -p 2200 -i ./grader`
where grader is the text file containing the local key to connect as grader user.

## Configuration Steps
### 1. Get update on installed packages and upgrade them

```
sudo apt-get update
sudo apt-get upgrade
```

### 2. Allow TCP port 2200 in Amazon Lightsail firewall configuration as we are using non default port of 2200 (instead of port 22) to SSH to the server.

### 3. Change SSH port to 2200
```
sudo nano /etc/ssh/sshd_config
# change the line 'Port 22' to 'Port 2200', and save the file
```

### 4. Configure Firewall (UFW) to allow ports 2200 (SSH), 80 (HTTP) and 123 (NTP) only.
```
# close all incoming ports
sudo ufw default deny incoming
# open all outgoing ports
sudo ufw default allow outgoing
# open ssh port
sudo ufw allow 2200/tcp
# open http port
sudo ufw allow 80/tcp
# open ntp port
sudo ufw allow 123/udp
# turn on firewall
sudo ufw enable
```

### 5. Create new user named *grader* and grant the sudo permissions
```
# create new user
sudo adduser grader

# copy 90-cloud-init-users file in the sudoers.d and rename as grader
sudo cp /etc/sudoers.d/90-cloud-init-users /etc/sudoers.d/grader
# change ubuntu as grader in the file
sudo nano /etc/sudoers.d/grader
```

### 6. Setup public key for grader and enforce key-based authentication
```
# in the local machine, generate the public key
ssh-keygen -f ~/.ssh/grader
```
Back on grader's terminal, put the public key in the grader user's .ssh folder on the server.
```
mkdir /home/grader/.ssh
nano /home/grader/.ssh/authorized_keys
# copy public key from the local machine and paste into the authorized_keys file on the server, and save
```

### 7. Disable remote login for *root* user
```
sudo nano /etc/ssh/sshd_config
# set PermitRootLogin to no, and save the file

# restart ssh service
sudo service ssh restart
```

### 8. Configure timezone to UTC
```
sudo dpkg-reconfigure tzdata
# choose 'None of the above' and then select 'UTC'
```

### 9. Install the Apache web server and WSGI module
```
sudo apt-get install apache2 libapache2-mod-wsgi
```

### 10. Install and configure PostgreSQL
```
sudo apt-get install PostgreSQL
# check that remote connections are not allowed in PostgreSQL config file
sudo nano /etc/postgresql/9.3/main/pg_hba.config
```

### 11. Create a user named catalog
```
# create a linux user named catalog
sudo adduser catalog

# create a PostgreSQL role named catalog and a database named catalog
sudo -u postgres -i
postgres:~$ creatuser catalog
postgres:~$ createdb catalog
postgres:~$ psql
postgres=# ALTER DATABASE catalog OWNER TO catalog;
postgres=# ALTER USER catalog WITH PASSWORD 'catalog';
postgres=# \q
postgres:~$ exit
```

### 12. Install git and clone the [Item Catalog](https://github.com/marcogregorius/item_catalog) project.
```
sudo apt-get install git
cd /var/www/
git clone https://github.com/marcogregorius/item_catalog
```

### 13. Install the remaining python libraries required
```
sudo apt-get install python-pip python-dev python-psycopg2
sudo pip install sqlalchemy
sudo pip install flask
sudo pip install oauth2client
sudo pip install httplib
sudo pip install json
sudo pip install requests
sudo pip install datetime
sudo pip install uuid
sudo pip install hashlib
sudo pip install functools
```

### 14. Change project.py file name to `__init__.py`
```
sudo mv /var/www/item_catalog/project.py /var/www/item_catalog/__init__.py
```

### 15. Configure the web app to connect to PostgreSQL instead of SQLite
Locate the following line in *database_setup.py, populate_categories.py and* `__init__.py`:
```
engine = create_engine('sqlite:///catalognew.db')
```
And change with below line:
```
engine = create_engine('postgresql://catalog:catalog@localhost/catalog')
```

### 16. Replace all lines in __init__.py having this command to load the client_secrets.json:
```
CLIENT_ID = json.loads(
    open('client_secrets.json', 'r').read())['web']['client_id']
```
with below line:
```
APP_PATH = '/var/www/item_catalog/'
CLIENT_ID = json.loads(
    open(APP_PATH + 'client_secrets.json', 'r').read())['web']['client_id']
```
The change is required as Ubuntu regards the open method's path as absolute path, not relative path.

### 17. Initialize categories schema by running populate_categories.py
```
python populate_categories.py
```

### 18. Configure Apache to serve the web application using WSGI
```
# create the web app WSGI file
sudo nano /var/www/app.wsgi
```
Add the following lines to the file and save the file:
```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/")
sys.path.insert(5,"/usr/local/lib/python2.7/site-packages/")
from item_catalog import app as application
application.secret_key = 'super_secret_key'
```
Update the Apache configuration file to serve the web application with WSGI.
```
sudo nano /etc/apache2/sites-enabled/000-default.conf
```
Add the following line inside the <VirtualHost *:80> element, and save the file.
```
WSGIScriptAlias / /var/www/app.wsgi
```

### 19. Restart Apache
```
sudo service apache2 restart
```

### 20. Test the web app
- Browse to the IP address http://54.255.161.210
- If the web is showing error, use `sudo tail /var/log/apache2/error.log` to debug.

## Create DNS server
- Subscribe to any DNS service provider
- Add in http://54.255.161.210 as the default IP address when visiting the DNS name
- Check out my website with DNS configured at http://catalog.marcogreg.com
