# Portfolio Setup

This is a guide for making your portfolio accessible from your server to the internet. I am gonna use my own [portfolio](https://github.com/Wacky-sama/Portfolio.git) as an example for this guide. Visit my [Cloudflare Tunnel Guide](/docs/cloudflare-tunnel-setup-guide.md).

---

## Step 1: Move your portfolio to **/var/www/** or **/srv**, your choice

```bash
sudo mv Portfolio /var/www/
```

---

## Step 2: Change the owner

```bash
sudo chown -R www-data:www-data /var/www/Portfolio
```

---

## Step 3: Install dependencies, and build the React app

1. Install Node.js and npm

```bash
sudo apt update
sudo apt install nodejs npm -y
node -v
npm -v
```

2. Install project dependencies

Go to your project folder:

```bash
cd /var/www/Portfolio
sudo -u www-data mkdir -p /var/www/Portfolio/.npm
sudo -u www-data npm install --cache /var/www/Portfolio/.npm
sudo -u www-data npm install
```

This will install vite and all packages defined in package.json.

3. Build the React app

```bash
sudo -u www-data npm run build
```

This should create a **dist** folder (Vite uses dist instead of build like CRA).

4. Serve the files

You can copy them to your web root:

```bash
sudo cp -r dist/* /var/www/Portfolio/
sudo chown -R www-data:www-data /var/www/Portfolio
```

---

## Step 4: Set up Apache

Create a site config:

```bash
sudo nano /etc/apache2/sites-available/portfolio.conf
```

Paste this and change what is necessary:

```bash
<VirtualHost *:80>
    ServerAdmin your-email@gmail.com
    DocumentRoot /var/www/Portfolio/dist
    ServerName yourdomain.com
    ServerAlias www.yourdomain.com

    <Directory /var/www/Portfolio/dist>
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

YOU ARE DONE! Enjoy!
