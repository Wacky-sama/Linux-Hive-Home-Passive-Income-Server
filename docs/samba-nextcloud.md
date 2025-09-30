# Samba and NextCloud Guide

### Installing Samba, NextCloud, Php, and MariaDB

Samba

```bash
sudo apt update
sudo apt install samba -y
```

NextCloud

```bash
wget https://download.nextcloud.com/server/releases/latest.zip
```

Extract into your web server folder:

```bash
sudo apt install unzip -y
sudo unzip latest.zip -d /var/www/
sudo mv /var/www/nextcloud /var/www/html/
```

Make Apache serve it:

```bash
sudo chown -R www-data:www-data /var/www/html/nextcloud
```

PHP

```bash
sudo apt update
sudo apt install apache2 libapache2-mod-php php-gd php-json php-mysql php-curl php-mbstring php-intl php-imagick php-xml php-zip php-bz2 php-gmp -y
```

MariaDB

```bash
sudo apt install mariadb-server -y
sudo mariadb-secure-installation
```

**Note:** Enter a root password, you answer three **'no'**, and four **'yes'**

Create a database + user:

```bash
CREATE DATABASE nextcloud;
CREATE USER 'ncuser'@'localhost' IDENTIFIED BY 'yourpassword';
GRANT ALL PRIVILEGES ON nextcloud.* TO 'ncuser'@'localhost';
FLUSH PRIVILEGES;
```

### Samba Config

Example Samba config (/etc/samba/smb.conf):

```bash
[shared]
   path = /srv/shared
   browseable = yes
   read only = no
   guest ok = no
   valid users = wacky
```

Then restart Samba:

```bash
sudo systemctl restart smbd
```

### Configure Apache for Nextcloud
Create a config file:
```bash
sudo nano /etc/apache2/sites-available/nextcloud.conf
```
Paste this:
```bash
<VirtualHost *:80>
    DocumentRoot /var/www/html/nextcloud/
    ServerName your-server-ip-or-domain

    <Directory /var/www/html/nextcloud/>
        Require all granted
        AllowOverride All
        Options FollowSymLinks MultiViews
    </Directory>
</VirtualHost>
```
Enable site + modules:
```bash
sudo a2ensite nextcloud.conf
sudo a2enmod rewrite headers env dir mime
sudo systemctl reload apache2
```