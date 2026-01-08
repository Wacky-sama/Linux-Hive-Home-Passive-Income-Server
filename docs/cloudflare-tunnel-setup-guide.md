# Cloudflare Tunnel Setup Guide

Before you follow the steps, make sure have these:

1. Domain(Buy on GoDaddy or which to the one you prefer)
2. [Cloudflare Account](https://dash.cloudflare.com/login)

---

## Step 1: Install **cloudflared**

Run this on your server:

```bash
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee cloudflare-main.gpg >/dev/null

echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/cloudflared.list

sudo apt update && sudo apt install cloudflared -y
```

If you encountered an error like this:

```bash
Hit:1 http://deb.debian.org/debian trixie InRelease Hit:2 http://deb.debian.org/debian trixie-updates InRelease Hit:3 http://security.debian.org/debian-security trixie-security InRelease Hit:4 https://download.docker.com/linux/debian trixie InRelease Ign:5 https://pkg.cloudflare.com/cloudflared trixie InRelease Err:6 https://pkg.cloudflare.com/cloudflared trixie Release 404 Not Found [IP: 104.18.0.118 443] Error: The repository 'https://pkg.cloudflare.com/cloudflared trixie Release' does not have a Release file. Notice: Updating from such a repository can't be done securely, and is therefore disabled by default. Notice: See apt-secure(8) manpage for repository creation and user configuration details.
```

Do this to fix the error:

```bash
# Remove the broken repo entry
sudo rm /etc/apt/sources.list.d/cloudflared.list

# Add bookworm repo
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared bookworm main" | sudo tee /etc/apt/sources.list.d/cloudflared.list

# Update and install
sudo apt update && sudo apt install cloudflared -y
```

What changed? I changed **$(lsb_release -cs)** to **bookworm**.

Now check:

```bash
cloudflared --version
```

---

## Step 2: Log in to Cloudflare

This links the tunnel to your account:

```bash
cloudflared tunnel login
```

It’ll open a URL. Copy-paste it into your browser, choose your domain, approve.

---

## Step 3: Create the tunnel

**NOTE**: Change **tunnel_name** to your liking.

```bash
cloudflared tunnel create tunnel_name
```

This prints out a Tunnel ID — Cloudflare remembers that for you.

Move the file to root:

```bash
sudo mkdir -p /root/.cloudflared

sudo cp /home/<username>/.cloudflared/<TUNNEL-ID>.json /root/.cloudflared/

sudo chown root:root /root/.cloudflared/<TUNNEL-ID>.json

sudo chmod 600 /root/.cloudflared/<TUNNEL-ID>.json
```

---

## Step 4: Create a directory

```bash
sudo mkdir -p /etc/cloudflared
```

---

## Step 5: Create the config file

```bash
sudo nano /etc/cloudflared/config.yml
```

Paste this config:

```bash
tunnel: tunnel_name
credentials-file: /root/.cloudflared/<TUNNEL-ID>.json

ingress:
  - hostname: yourdomain.com
    service: http://localhost:80

  - service: http_status:404
```

---

## Step 6: Create the DNS route

```bash
cloudflared tunnel route dns nextcloud cloud.yourdomain.com
```

This auto-creates the CNAME in Cloudflare.

---

## Step 7: Run it as a service

```bash
sudo cloudflared service install
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
```

Verify:

```bash
sudo systemctl status cloudflared
```

---

Visit my [Portfolio Setup Guide](/docs/portfolio-setup-guide.md).
