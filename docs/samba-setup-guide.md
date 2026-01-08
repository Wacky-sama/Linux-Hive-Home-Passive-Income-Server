# Samba Setup Guide

This guide shows how to install and configure Samba on a Debian/Ubuntu server to share files and folders on your local network. It covers package installation, opening necessary ports, creating shares, adding Samba users, and enabling the service.

---

## Step 1: Update your packages then install the samba:

```bash
sudo apt update

sudo apt install samba samba-common-bin -y
```

---

## Step 2: Open these ports for Samba:

```bash
sudo ufw allow 139/tcp
sudo ufw allow 445/tcp
sudo ufw allow 137/udp
sudo ufw allow 138/udp
```

Quick breakdown: 445 is the main SMB port, 139 is for NetBIOS sessions, and 137/138 handle NetBIOS name services.

---

## Step 3: Edit the Samba config:

```bash
sudo nano /etc/samba/smb.conf
```

Add something like this at the bottom (customize to your needs):

```bash
[SharedFolder]
   path = /path/to/your/folder
   browseable = yes
   read only = no
   guest ok = no
   valid users = your_user
```

Create a Samba password for your user:

```bash
sudo smbpasswd -a your_user
```

Enable and start the services:

```bash
sudo systemctl enable smbd
sudo systemctl start smbd
sudo systemctl enable nmbd
sudo systemctl start nmbd
```

Check if it's running:

```bash
sudo systemctl status smbd
```

---

DONE!
