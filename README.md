# vm-server

Baseline installation of a Linux distribution on a virtual machine, secured and prepared for hosting web apps
```bash
# as root:
# update system
apt-get update && sudo apt-get upgrade
# add new user and add him to sudo group
adduser grader sudo
# give a password to grader
passwd grader
# setup ssh for the new user
mkdir /home/grader/.ssh
chown -R grader:grader /home/grader/.ssh/
chmod 700 /home/grader/.ssh/
# the permissions remains unchanged upon copy (600)
cp .ssh/authorized_keys /home/grader/.ssh/

# cron job to update every week
crontab -e
# add this line to the end and save
0 5 * * 1 sudo apt-get -qq update && sudo apt-get -y upgrade
# check cron jobs
crontab -l
```

ssh config, file: /etc/ssh/sshd_config

```bash
PORT 2200
PermitRootLogin no
PubkeyAuthentication yes

```
installed firewall

### Install and confgure firewall
```bash
$ sudo apt-get install ufw
```

```bash
$ sudo ufw default deny
```
allow 2200 which is ssh
```bash
$ sudo ufw allow 2200
```
allow 80 for apache
```bash
$ sudo ufw allow 80
```
allow ntp
```bash
$ sudo ufw allow 123
```
deny connections from an IP address that has attempted to initiate 6 or more connections in the last 30 seconds
```bash
$ sudo ufw limit 2200
```
Disable remote ping  
Change ACCEPT to DROP in the following lines:  
```bash
/etc/ufw/before.rules
-------------------------------------------------------------------------
# ok icmp codes
-A ufw-before-input -p icmp --icmp-type destination-unreachable -j ACCEPT
-A ufw-before-input -p icmp --icmp-type source-quench -j ACCEPT
-A ufw-before-input -p icmp --icmp-type time-exceeded -j ACCEPT
-A ufw-before-input -p icmp --icmp-type parameter-problem -j ACCEPT
-A ufw-before-input -p icmp --icmp-type echo-request -j ACCEPT
```
finally enable firewall
```bash
$ sudo ufw enable
```

### Install necessary programs

```bash
apt-get -y install git apache2 libapache2-mod-wsgi
apt-get -y install libpq-dev python-dev
apt-get -y install postgresql
apt-get -y install python-sqlalchemy
apt-get -y install python-pip
pip install virtualenv
```

### Postgres config


```bash
# login as postgres user
sudo su - postgres
psql
```
```sql
-- Inside psql shell
-- create role with limited permissions.
CREATE ROLE catalog WITH login password;
-- create a role to access the db
CREATE ROLE access_catalog;
CREATE DATABASE catalog WITH OWNER access_catalog;
\c catalog
REVOKE ALL ON SCHEMA public FROM public;
GRANT ALL ON SCHEMA public TO access_role;
RESET ROLE;
-- give catalog db access to role catalog
GRANT access_catalog TO login_catalog;
```

edit postgresql access,  


file: `/etc/postgresql/9.3/main/pg_hba.conf`  

- add this line.
```bash
# TYPE   DATABASE            USER              ADDRESS              METHOD
local    catalog             catalog                                md5
```
- ensure remote connections aren't allowed (look for type and address).

### Apache config

- create a vhosts file for our application.

```bash
# example file
<VirtualHost *>
    #ServerName example.com
    ServerAdmin admin@email.com
    WSGIDaemonProcess pytalog user=user group=group threads=5
    WSGIScriptAlias / /var/www/pytalog/pytalog.wsgi

    Alias /static /var/www/pytalog/pytalog/static
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined

    <Directory /var/www/pytalog/pytalog>
        WSGIProcessGroup pytalog
        WSGIApplicationGroup %{GLOBAL}
        WSGIScriptReloading On
        Order deny,allow
        Allow from all
    </Directory>
</VirtualHost>
```

- git clone the application in the vm and copy the directory in `/var/www/pytalog`

- create a `pytalog.wsgi`

```python
# example file
import sys
activate_this = '/var/www/pytalog/env/bin/activate_this.py'
# activate virtual enviroment
execfile(activate_this, dict(__file__=activate_this))
sys.path.insert(0,"/var/www/pytalog/")

from pytalog import app as application
if not application.debug:
   import logging
   from logging import Formatter
   from logging.handlers import RotatingFileHandler
   file_handler = RotatingFileHandler('/var/www/pytalog/logs/app.log', maxBytes=1000, backupCount=10)
   file_handler.setFormatter(Formatter(
       '%(asctime)s %(levelname)s: %(message)s '
       '[in %(pathname)s:%(lineno)d]'
   ))
   file_handler.setLevel(logging.WARNING)
   application.logger.addHandler(file_handler)
```
Add `.htacces` in to the `.git` directory  
```bash
.htaccess
------------------
Order allow,deny
Deny from all

```
Subscribed and setup newrelic to monitor the server.

Url of the application  
http://ec2-52-24-182-116.us-west-2.compute.amazonaws.com/

vms ip `52.24.182.116` port `2200`

### resources:


- securing postgresql https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps
- ufw firewall https://wiki.archlinux.org/index.php/Uncomplicated_Firewall
- flask deployment http://flask.pocoo.org/docs/0.10/deploying/
