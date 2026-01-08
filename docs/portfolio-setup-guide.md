# Portfolio Setup

This is a guide for making your portfolio accessible from your server to the internet. I am gonna use my own [portfolio](https://github.com/Wacky-sama/Portfolio.git) as an example for this guide.

---

## Step 1: Move your portfolio to /var/www/ or /srv, your choice

```bash
sudo mv Portfolio /var/www/
```

---

## Step 2: Change the owner

```bash
sudo chown -R www-data:www-data /var/www/Portfolio
```

---

## Step 3: Set up Apache

Create a site config:

```bash
sudo nano /etc/apache2/sites-available/portfolio.conf
```

Paste this and change what is necessary:

```bash
<VirtualHost *:80>
    ServerAdmin your-email@gmail.com
    DocumentRoot /var/www/Portfolio
    ServerName yourdomain.com
    ServerAlias www.yourdomain.com

    <Directory /var/www/Portfolio>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/portfolio_error.log
    CustomLog ${APACHE_LOG_DIR}/portfolio_access.log combined
</VirtualHost>
```

Enable the site:

```bash
sudo a2ensite portfolio.conf

sudo systemctl restart apache
```

---
