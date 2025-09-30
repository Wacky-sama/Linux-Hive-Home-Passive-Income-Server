# Samba and NextCloud Guide

### Installing Samba and NextCloud
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