# Portainer Guide

This guide shows how to setup your Portainer.

---

NOTE! 

Container name: `portainer`
Docker volume: `portainer_data`

## Step 1: Create directory

```bash
sudo mkdir -p /opt/portainer
```

## Step 2: Create Volume

```bash
docker volume create portainer_data
```

Verify:
```bash
docker volume ls | grep portainer
```

## Step 3: Run Portainer

```bash
docker run -d --name portainer --restart=always -p 8000:8000 -p 9443:9443 -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest
```

What each part does
- `--restart=always` > survives reboots
- `docker.sock` > allows Portainer to control Docker
- `/data` > persistent config, users, stacks

Verify it’s running:

```bash
docker ps | grep portainer
```

## Step 4: Web Setup

Open in browser:

```bash
https://SERVER_IP:9443
```

Initial steps

1. Create admin user
2. Choose Local environment
3. Click Connect

You’ll land on the dashboard showing:
- Containers
- Images
- Volumes
- Networks

## Step 5: Post-Install

Setup Firewall

```bash
sudo ufw allow from 192.168.xx.xx/24 to any port 9443
sudo ufw deny 9443
```

---

### Congratulations! You have a GUI for managing your Docker containers now! Keep it up, SysAdmin!