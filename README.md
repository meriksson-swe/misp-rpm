# misp-rpm
Instructions for installation of MISP on Rocky/Alma Linux 8

# Install MISP from RPM packages

## Installation instructions:

- install Rocky/Alma Linux 8 minimal system, install license
- update system to latest updates

## install epel, remi and misp repositories

```
dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
dnf install https://rpms.remirepo.net/enterprise/remi-release-8.rpm
dnf install https://repo.misp-project.ch/yum/misp8/misp-release-latest.el8.noarch.rpm
```

## install misp and misp-python-virtualenv
```
dnf install misp misp-python-virtualenv
```

## install MariaDB, if you want to use an external DB, only MariaDB-client is needed
```
dnf install MariaDB-client MariaDB-server
```

## configuration
- configure mariadb to your needs

```
# enable mariadb startup
systemctl enable mariadb.service
systemctl start mariadb.service

# secure the installation, set a reasonable root password
mariadb-secure-installation

# install MISP DB schema
mariadb -u root -p [YOUR MYSQL PASSWORD]

# replace XXXXXXXXX with a reasonable password for misp
MariaDB [(none)]> create database misp;
MariaDB [(none)]> grant usage on *.* to misp@localhost identified by 'XXXXXXXXX';
MariaDB [(none)]> grant all privileges on misp.* to misp@localhost;
MariaDB [(none)]> exit

cd /var/www/MISP

# Import the empty MySQL database from MYSQL.sql
mariadb -u misp -p misp < INSTALL/MYSQL.sql
```

- configure MISP

```
cd /var/www/MISP/app/Config

# set DB details in database.php, use your XXXXXXXXX password from above
# set 'login' => 'misp'
# set 'password' => 'XXXXXXXXX'

# set baseurl in config.php
# set python_bin => '/var/www/cgi-bin/misp-virtualenv/bin/python3'
# In array 'SimpleBackgroundJobs'
# set enabled => true
# replace YYYYYYYYYY with a reasonable password for supervisord
# set supervisor_password => 'YYYYYYYYYY'

# set owner and selinux context
chown apache:apache /var/www/MISP/app/Config/config.php
chcon -t httpd_sys_rw_content_t /var/www/MISP/app/Config/config.php

```
- configure supervisord
```
cd /etc/supervisord.d

# In misp-workers.ini add scheduler at second line
[group:misp-workers]
programs=default,email,cache,prio,update, scheduler

# Add this block at the end of the file
[program:scheduler]
directory=/var/www/MISP
command=/var/www/MISP/app/Console/cake scheduler_worker
process_name=%(program_name)s_%(process_num)02d
numprocs=1
autostart=false
autorestart=true
redirect_stderr=false
stderr_logfile=/var/www/MISP/app/tmp/logs/misp-workers-errors.log
stdout_logfile=/var/www/MISP/app/tmp/logs/misp-workers.log
user=apache

# In /etc/supervisord.conf, uncomment these lines and set password to YYYYYYYYYY as above
[inet_http_server]         ; inet (TCP) server disabled by default
port=127.0.0.1:9001        ; (ip_address:port specifier, *:port for all iface)
username=user              ; (default is no username (open server))
password=123               ; (default is no password (open server))

# change to
# user=supervisor
# password=YYYYYYYYYY
``` 

- configure php

all php settings are done in ```/etc/opt/remi/php83/php.ini```

- link php
```
ln -s /bin/php83 /bin/php
```

- start redis

```
# enable redis at startup
systemctl enable redis
systemctl start redis
```

- configure httpd

```
# make sure apache is the owner och /var/www/MISP
# chown -R apache:apahce /var/www/MISP

# remove /etc/httpd/conf.d/ssl.conf
# create /etc/httpd/conf.d/misp-ssl.conf with the following content

Listen 443 https

<VirtualHost *:443>
    ServerAdmin admin@admin.test
    ServerName misp.local
    DocumentRoot /var/www/MISP/app/webroot

    <Directory /var/www/MISP/app/webroot>
          Options -Indexes
          AllowOverride all
  	        Require all granted
          Order allow,deny
          allow from all
     </Directory>

     SSLEngine On
     SSLCertificateFile /etc/pki/tls/certs/localhost.crt
     SSLCertificateKeyFile /etc/pki/tls/private/localhost.key

     LogLevel warn
     ErrorLog /var/log/httpd/misp.error.log
     CustomLog /var/log/httpd/misp.access.log combined
     ServerSignature Off
     Header set X-Content-Type-Options nosniff
     Header set X-Frame-Options DENY
     SSLCipherSuite HIGH:!aNULL:!SHA1:!MD5:!DHE:!DH:!ADH
 </VirtualHost>

# enable apache at startup
systemctl enable httpd
systemctl start httpd
```

```
# enable php-fpm at startup
systemctl enable php83-php-fpm
systemctl start php83-php-fpm
```

```
# enable supervisord at startup
systemctl enable --now supervisord
systemctl start supervisord

If not all workers are running when you check status
# supervisorctl status

Then run these and check again 
# supervisorctl reread
# supervisorctl update
# supervisorctl restart misp-workers:*
```


- open firewall

```
# open firewall for http and https
firewall-cmd --permanent --zone=public --add-service http
firewall-cmd --permanent --zone=public --add-service https
systemctl restart firewalld
```

## install and enable misp-modules
```
dnf install misp-modules
# enable misp-modules at startup
systemctl enable misp-modules
systemctl start misp-modules

I had problems installing misp-modules.
1. Missing dependency poppler-cpp. Downloaded it from https://www.rpmfind.net/linux/almalinux/8.10/PowerTools/x86_64/os/Packages/poppler-cpp-20.11.0-11.el8.x86_64.rpm and installed it locally with
    # dnf localinstall ./poppler-cpp-20.11.0-11.el8.x86_64.rpm
2. Unsigned package misp-modules-3.0.5. Installed earlier version instead
    # dnf install misp-modules-3.0.3
```

- **reboot the host to make sure all services are started correctly**

