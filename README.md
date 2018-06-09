# Linux Server Configuration

This README shows how to set up this [Item Catalog Website](https://github.com/marcogregorius/item_catalog) on a Ubuntu Server hosted with Amazon Lightsail service. Includes installing updates, securing the server from a number of attack vectors, and installing/configuring web and database servers.

## Server Information
- IP Address: 54.255.161.210
- SSH port: 2200
- SSH connection command as `grader`:

`ssh grader@54.255.161.210 -p 2200 -i ./grader`
where grader is the text file containing the local key to connect as grader user.

## Configuration Steps
1. Get update on installed packages and upgrade them
```
sudo apt-get update
sudo apt-get upgrade
```

2. Allow TCP port 2200 in Amazon Lightsail firewall configuration as we are using non default port of 2200 (instead of port 22) to SSH to the server.

3. Change SSH port to 2200
```
sudo nano /etc/ssh/sshd_config
# change the line 'Port 22' to 'Port 2200', and save the file
```

4. Configure Firewall (UFW) to allow ports 2200 (SSH), 80 (HTTP) and 123 (NTP) only.
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

5. Create new user named *grader* and grant the sudo permissions
```
# create new user
sudo adduser grader

# copy 90-cloud-init-users file in the sudoers.d and rename as grader
sudo cp /etc/sudoers.d/90-cloud-init-users /etc/sudoers.d/grader
# change ubuntu as grader in the file
sudo nano /etc/sudoers.d/grader
```

6. Setup public key for grader and enforce key-based authentication
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

7. Disable remote login for *root* user
```
sudo nano /etc/ssh/sshd_config
# set PermitRootLogin to no, and save the file

# restart ssh service
sudo service ssh restart
```

8. Configure timezone to UTC
```
sudo dpkg-reconfigure tzdata
# choose 'None of the above' and then select 'UTC'
```

9. Install the Apache web server and WSGI module
```
sudo apt-get install apache2 libapache2-mod-wsgi
```

10. Install and configure PostgreSQL
```
sudo apt-get install PostgreSQL
# check that remote connections are not allowed in PostgreSQL config file
sudo nano /etc/postgresql/9.3/main/pg_hba.config
```

11. Create a user named catalog
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

12. Install git and clone the [Item Catalog](https://github.com/marcogregorius/item_catalog) project.
```
sudo apt-get install git
cd /var/www/
git clone https://github.com/marcogregorius/item_catalog

