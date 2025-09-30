# NextCloud and Samba Setup Guide

## NextCloud Setup

Step 1: Update your server

```bash
sudo apt update && sudo apt upgrade -y
```

Step 2: Install Required Packages

```bash
sudo apt install -y apache2 mariadb-server libapache2-mod-php php-gd php-json php-mysql php-curl php-mbstring php-intl php-imagick php-xml php-zip unzip wget
```

Step 3: Secure MariaDB
Run the secure installation:

```bash
sudo mariadb-secure-installation
```

- Set a root password
- Remove anonymous users
- Disallow remote root login
- Remove test database

Step 4: Create Database for Nextcloud
Login to MariaDB:

```bash

sudo mysql -u root -p

```

Then inside MariaDB:

```bash
CREATE DATABASE nextcloud;
CREATE USER 'nextclouduser'@'localhost' IDENTIFIED BY 'StrongPasswordHere';
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextclouduser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Step 5: Download Nextcloud

```bash
cd /var/www/
wget https://download.nextcloud.com/server/releases/latest.zip
sudo unzip latest.zip
```

Step 6: Set permissions
**Note:** This is still inside **/var/www/**

```bash
sudo chown -R www-data:www-data nextcloud
sudo chmod -R 755 nextcloud
```

Step 7: Configure Apache

```bash
sudo nano /etc/apache2/sites-available/nextcloud.conf
```

Paste this config:

```bash
<VirtualHost *:80>
    ServerAdmin your-email-address
    DocumentRoot /var/www/nextcloud
    ServerName your-server-ip

    <Directory /var/www/nextcloud/>
        Options FollowSymLinks MultiViews
        AllowOverride All
        Require all granted
        <IfModule mod_dav.c>
            Dav off
        </IfModule>
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/nextcloud_error.log
    CustomLog ${APACHE_LOG_DIR}/nextcloud_access.log combined
</VirtualHost>
```

Enable required modules and the site:

```bash
sudo a2ensite nextcloud.conf
sudo a2enmod rewrite headers env dir mime
sudo systemctl restart apache2
```

Step 8: Access Nextcloud
Open a browser and go to:
http://YOUR_SERVER_IP/
You’ll see the Nextcloud setup page:

- Create an admin account
- Enter the database user (nextclouduser), password, and database name (nextcloud)
- Leave host as localhost
  Click Install.
