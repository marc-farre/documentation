---
id: server-setup
title: Server Setup (Linux)
---

This document covers a basic server configuration for a HumHub. 
The operating system used here is ``Linux`` with the distribution ``Debian Buster 10``.

HumHub is being installed into the ``/var/www/humhub`` directory here and runs with the user/group ``www-data``.

In the example configuration the URL https://temp.humhub.dev is used. 
This must be changed according to your address.

## Database

In this section we will create a database with the following values: 

- **Datebase:** humhub_prod_db
- **User:** humhub_prod
- **Password:** change-me

Of course these values, especially the password, should be changed according to your environment.


### Installation


Install and configure MariaDB Server using Debian packages.

```bash
apt update
apt install mariadb-server mariadb-client automysqlbackup
mysql_secure_installation
```

### Create Database Schema

Open MySQL console

```bash
mysql -u root -p
```

Create database user

```sql
mysql> CREATE USER 'humhub_prod' IDENTIFIED BY 'change-me';
``` 

Create a MySQL Database, e.g.:

```sql
mysql> CREATE DATABASE `humhub_prod_db` CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
mysql> GRANT ALL ON `humhub_prod_db`.* TO `humhub_prod`@localhost IDENTIFIED BY 'change-me';
mysql> FLUSH PRIVILEGES;
```

:::important
Do not forget to change the `change-me` placeholder!
:::


## PHP

### Installation 

```bash
apt update
apt install php php-cli \
	php-imagick php-curl php-bz2 php-gd php-intl \
	php-mysql php-zip php-apcu-bc php-apcu php-xml php-ldap

```

### Configuration

> TBD: PHP ini tweaks

## WebServer


### Preparation


Installation Certbot and obtain SSL certificates for the portal.

```bash
apt install certbot
certbot certonly --standalone -d temp.humhub.dev
```

### Apache

Install ``Apache 2.4`` core and modules from Debian package repositories.

```
apt update
apt install apache2 \
	libapache2-mod-xsendfile \
	php-fpm 
```

Create configuration file ``/etc/apache2/site-available/humhub.conf`` with following content.


```apacheconf
<VirtualHost *:443>
  ServerName temp.humhub.dev
  ServerAdmin root@temp.humhub.dev

  SSLEngine on
  SSLCertificateFile    	/etc/letsencrypt/live/temp.humhub.dev/cert.pem
  SSLCertificateKeyFile 	/etc/letsencrypt/live/temp.humhub.dev/privkey.pem
  SSLCertificateChainFile 	/etc/letsencrypt/live/temp.humhub.dev/fullchain.pem

  DocumentRoot /var/www/humhub

  <Directory /var/www/humhub/>
     Options -Indexes -FollowSymLinks
     AllowOverride All
  </Directory>

  <DirectoryMatch "/var/www/humhub/(\.|protected|themes/\w+/views|uploads/file)">
     Order Deny,Allow
     Deny from all
  </DirectoryMatch>

  <FilesMatch "^\.">
     Order Deny,Allow
     Deny from all
  </FilesMatch>
</VirtualHost>

<VirtualHost *:80>
   ServerName temp.humhub.dev
   Redirect / https://temp.humhub.dev
</VirtualHost>
```

Enable the virtual host and additional Apache2 configurations and required modules.

```bash
a2enmod ssl rewrite headers proxy_fcgi setenvif
a2enconf php7.3-fpm
a2ensite humhub

systemctl reload apache2
```


### NGINX


```
apt update
apt install nginx \
	php-fpm
```


```conf
server {
    listen 80;
    listen [::]:80;

    server_name temp.humhub.dev;
    return 301 https://$server_name:443$request_uri;
}

server {
	listen 443 ssl;
	listen [::]:443 ssl;
	
	root /var/www/humhub;

	server_name temp.humhub.dev;
	
	include snippets/letsencrypt.conf;
    ssl_certificate /etc/letsencrypt/live/temp.humhub.dev/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/temp.humhub.dev/privkey.pem;

	charset utf-8;

	location / {
		index  index.php index.html ;
		try_files $uri $uri/ /index.php?$args;
	}

	location ~ ^/(protected|framework|themes/\w+/views|\.|uploads/file) {
		deny all;
	}

	location ~ \.php {
		fastcgi_split_path_info  ^(.+\.php)(.*)$;

		#let yii catch the calls to unexising PHP files
		set $fsn /index.php;
		if (-f $document_root$fastcgi_script_name){
				set $fsn $fastcgi_script_name;
		}

  		fastcgi_pass unix:/var/run/php/php7.3-fpm.sock;
		include fastcgi_params;
		fastcgi_param  SCRIPT_FILENAME  $document_root$fsn;
	}
}
```


## Mail 

If e-mails should be sent directly from this server, an SMTP server must be installed.

### Postfix

```
apt update
apt install postfix
```

Please choose ``Internet Site`` as installation type.