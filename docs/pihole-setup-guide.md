# Pi-hole Setup Guide

This is a guide for setting up Pi-Hole via Docker.

---

## Step 1: Create a directory for Pi-hole

```bash
sudo mkdir -p ~/docker/pihole
cd ~/docker/pihole
```

---

## Step 2: Create docker-compose.yml

```bash
sudo nano docker-compose.yml
```

Enable port 53 in UFW:

```bash
sudo ufw allow from 192.168.0.0/16 to any port 53
```

Check if port 53 & 80 already listening or not.

```bash
sudo ss -tulp | grep :53

sudo ss -tulp | grep :80
```

If they returned nothing, you're good to go, but if they returned something, change the ports like the example below:

**NOTE:** Change this `FTLCONF_webserver_api_password: "change-this-now"` to your liking of password. You will use that for logging in to the Web UI later.

Paste this:

```bash
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest

    ports:
      # DNS (this is the important part)
      - "53:53/tcp"
      - "53:53/udp"

      # Pi-hole Web UI (moved off Apache ports)
      - "8081:80/tcp"

    environment:
      TZ: "Asia/Manila"

      # Web UI password
      FTLCONF_webserver_api_password: "change-this-now"

      # Required when using Docker bridge networking
      FTLCONF_dns_listeningMode: "ALL"

      # Disable HTTPS inside Pi-hole (Apache already handles this)
      FTLCONF_webserver_https: "false"

    volumes:
      - "./etc-pihole:/etc/pihole"
      # Enable ONLY if you really need custom dnsmasq configs
      # - "./etc-dnsmasq.d:/etc/dnsmasq.d"

    cap_add:
      - NET_ADMIN
      - SYS_NICE

    restart: unless-stopped
```

---

## Step 3: Bring it up

```bash
docker compose up -d
```

Check:

```bash
docker ps
docker logs pihole
```

You should see FTL started successfully with no port errors.

---

## Step 4: Access Pi-hole

Browser:

```bash
http://<SERVER-IP>:8081/admin
```

---

## Step 5: If you want to put in Cloudflare Tunnel do this:

Edit `/etc/cloudflare/config.yml`:

```bash
sudo nano /etc/cloudflare/config.yml
```

Paste this and change it accordingly:

```bash
tunnel: tunnel_name
credentials-file: /root/.cloudflared/<TUNNEL-ID>.json

ingress:
  - hostname: pihole.your-domain.com
    service: http://localhost:8081

  - service: http_status:404
```

Create the DNS route:

```bash
cloudflared tunnel route dns tunnel_name pihole.yourdomain.com
```

This auto-creates the CNAME in Cloudflare.

Then:

```bash
sudo systemctl restart cloudflared
sudo systemctl status cloudflared
```

---

You are done! You can access your Pi-hole outside your LAN.