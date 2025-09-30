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
- Switch to unix_socket authentication [Y/n] n
- Change the root password? [Y/n] n
- Remove anonymous users? [Y/n] Y
- Disallow root login remotely? [Y/n] Y
- Remove test database and access to it? [Y/n] Y
- Reload privilege tables now? [Y/n] Y

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
sudo wget https://download.nextcloud.com/server/releases/latest.zip
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

Disable the default site:

```bash
sudo a2dissite 000-default.conf
sudo systemctl restart apache2
```

Enable required modules and the site:

```bash
sudo a2ensite nextcloud.conf
sudo a2enmod rewrite headers env dir mime
sudo systemctl restart apache2
```

Step 8: Access Nextcloud

Open a browser and go to:

```bash
http://YOUR_SERVER_IP/
```

You’ll see the Nextcloud setup page:

- Create an admin account
- Enter the database user (nextclouduser), password, and database name (nextcloud)
- Leave host as localhost
  Click Install.

If you cannot create an account thru the Web, try this another method that I used:

```bash
sudo -u www-data php /var/www/nextcloud/occ maintenance:install \
  --database "mysql" \
  --database-name "nextcloud" \
  --database-user "nextclouduser" \
  --database-pass "StrongPasswordHere" \
  --admin-user "create_your_own_admin_user" \
  --admin-pass "create_your_own_admin_password"
```

Then you'll see this:

**Nextcloud was successfully installed**

After that, you can to your browser then refresh it. If you encounter this error:

Access through untrusted domain Please contact your administrator. If you are an administrator, edit the "trusted_domains" setting in config/config.php like the example in config.sample.php. Further information how to configure this can be found in the documentation.

Edit this file:

```bash
sudo nano /var/www/nextcloud/config/config.php
```

Look for the trusted_domains section

It should look something like this:

```bash
'trusted_domains' =>
array (
  0 => 'localhost',
),
```

Add your server’s IP or domain

```bash
'trusted_domains' =>
array (
  0 => 'localhost',
  1 => '192.168.1.100',
  2 => 'cloud.example.com',
),
```

Save and exit

In nano, press:

```bash
CTRL + O  (save)
ENTER
CTRL + X  (exit)
```

Restart Apache

```bash
sudo systemctl restart apache2
```

Now refresh your browser — the warning should disappear.
