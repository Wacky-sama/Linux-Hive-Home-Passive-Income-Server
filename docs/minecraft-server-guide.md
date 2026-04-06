# PaperMC + Geyser Server on Debian 13 (Trixie)

A complete guide for setting up a PaperMC Minecraft Java server with Geyser support for Bedrock/mobile clients on Debian 13.

- **OS:** Debian 13 (Trixie)
- **Java:** OpenJDK 21
- **Paper:** 1.21.11
- **Mode:** LAN / private (offline, no Microsoft auth)

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Install Java 21](#2-install-java-21)
3. [Create a Dedicated User](#3-create-a-dedicated-user)
4. [Download PaperMC](#4-download-papermc)
5. [First Run and EULA](#5-first-run-and-eula)
6. [Configure server.properties for LAN](#6-configure-serverproperties-for-lan)
7. [systemd Service](#7-systemd-service)
8. [UFW Firewall Rules](#8-ufw-firewall-rules)
9. [Install Geyser for Bedrock Support](#9-install-geyser-for-bedrock-support)
10. [Managing the Server](#10-managing-the-server)
11. [Updating PaperMC](#11-updating-papermc)

---

## 1. Prerequisites

Update your system and ensure `curl` and `jq` are installed.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl jq -y
```

Minimum recommended specs for a small private server:

| Resource | Minimum |
|----------|---------|
| RAM | 2 GB |
| CPU | 2 cores |
| Disk | 10 GB free |
| Network port | 25565/tcp (Java), 19132/udp (Bedrock) |

---

## 2. Install Java 21

PaperMC 1.21.x requires Java 21. On Debian 13, `default-jdk` resolves to Java 21 directly.

```bash
sudo apt install default-jdk -y 
```

Verify:

```bash
java -version
```

Expected output:

```
openjdk 21.0.x ...
OpenJDK Runtime Environment ...
OpenJDK 64-Bit Server VM ...
```

If the version shown is lower than 21, install it explicitly:

```bash
sudo apt install openjdk-21-jdk -y
```

---

## 3. Create a Dedicated User

Never run the server as root. Create a dedicated `minecraft` system user.

```bash
sudo useradd -r -m -d /opt/minecraft -s /bin/bash minecraft
sudo mkdir -p /opt/minecraft/server
sudo chown -R minecraft:minecraft /opt/minecraft
```

This user has no sudo privileges by design. Any system-level package installs must be done as your own user first.

---

## 4. Download PaperMC

PaperMC has migrated to a new API (`fill.papermc.io`). The old `api.papermc.io` v2 endpoint is being phased out and will be disabled on July 1, 2026. Use the correct method below.

Switch to the `minecraft` user and download the latest stable build:

```bash
sudo -u minecraft bash
cd /opt/minecraft/server

MC_VER="1.21.11"
USER_AGENT="your-server-name/1.0 (your-contact-or-domain)"

PAPER_URL=$(curl -s -H "User-Agent: $USER_AGENT" \
  "https://fill.papermc.io/v3/projects/paper/versions/${MC_VER}/builds" | \
  jq -r 'first(.[] | select(.channel == "STABLE") | .downloads."server:default".url)')

echo "Downloading: $PAPER_URL"
curl -L -H "User-Agent: $USER_AGENT" -o paper.jar "$PAPER_URL"
```

> **Note:** The PaperMC API requires a non-generic `User-Agent` header. Requests without one are rejected. Replace the placeholder with your own server name and contact.

Verify the download is a real jar and not an error response:

```bash
ls -lh paper.jar
```

It should be around 52 MB. If it is only a few hundred bytes, the download failed and you likely have a bad URL or missing User-Agent header.

Exit back to your regular user when done:

```bash
exit
```

---

## 5. First Run and EULA

Run the server once to generate all config files. It will stop immediately and ask you to accept the EULA.

```bash
sudo -u minecraft bash -c "cd /opt/minecraft/server && java -jar paper.jar --nogui"
```

Accept the EULA:

```bash
sudo -u minecraft sed -i 's/eula=false/eula=true/' /opt/minecraft/server/eula.txt
```

Run it again to fully initialize the world and generate all configuration files. This takes about 30 seconds on first run.

```bash
sudo -u minecraft bash -c "cd /opt/minecraft/server && java -jar paper.jar --nogui"
```

Wait for the line:

```
Done (Xs)! For help, type "help"
```

Then type `stop` and press Enter to shut it down cleanly before proceeding.

---

## 6. Configure server.properties for LAN

Edit `server.properties` to configure the server for private LAN use.

```bash
sudo -u minecraft nano /opt/minecraft/server/server.properties
```

Key settings to change for a private LAN server:

```properties
# Disable Microsoft/Mojang account verification.
# Players can join without owning a paid Java account.
# Only use this for trusted private/LAN servers.
online-mode=false

# Set your server name shown in the server list
motd=My Private Server

# Limit players to your RAM budget
max-players=10

# Lower view distance to reduce CPU and RAM usage.
# Mobile clients do not need more than 6-8 chunks.
view-distance=6

# Enable whitelist to restrict who can join (optional)
white-list=false
```

> **What online-mode=false does:** The server will no longer verify player accounts against Mojang's authentication servers. The Minecraft app itself still requires a Microsoft login to launch. This setting only affects server-side verification.

Save and exit with `Ctrl+X`, then `Y`, then `Enter`.

---

## 7. systemd Service

Create the systemd unit file to run the server as a managed background service.

```bash
sudo nano /etc/systemd/system/minecraft.service
```

Paste the following. Adjust `-Xmx` to leave headroom for your OS -- on a 4 GB machine, `2G` or `3G` is appropriate.

```ini
[Unit]
Description=PaperMC Minecraft Server
After=network.target

[Service]
Type=simple
User=minecraft
WorkingDirectory=/opt/minecraft/server
ExecStart=/usr/bin/java \
  -Xms1G \
  -Xmx2G \
  -XX:+UseG1GC \
  -XX:+ParallelRefProcEnabled \
  -XX:MaxGCPauseMillis=200 \
  -XX:+UnlockExperimentalVMOptions \
  -XX:+DisableExplicitGC \
  -XX:+AlwaysPreTouch \
  -XX:G1NewSizePercent=30 \
  -XX:G1MaxNewSizePercent=40 \
  -XX:G1HeapRegionSize=8M \
  -XX:G1ReservePercent=20 \
  -XX:G1HeapWastePercent=5 \
  -XX:G1MixedGCCountTarget=4 \
  -XX:InitiatingHeapOccupancyPercent=15 \
  -XX:G1MixedGCLiveThresholdPercent=90 \
  -XX:G1RSetUpdatingPauseTimePercent=5 \
  -XX:SurvivorRatio=32 \
  -XX:+PerfDisableSharedMem \
  -XX:MaxTenuringThreshold=1 \
  -jar paper.jar --nogui
ExecStop=/bin/kill -s SIGINT $MAINPID
Restart=on-failure
RestartSec=10s
TimeoutStopSec=60
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

The JVM flags used here are the Aikar flags -- industry-standard G1GC tuning specifically for Paper servers.

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable minecraft
sudo systemctl start minecraft
```

Watch the startup logs:

```bash
sudo journalctl -u minecraft -f
```

Press `Ctrl+C` to exit the log tail. The server keeps running in the background.

---

## 8. UFW Firewall Rules

Open the Minecraft Java port and the Geyser Bedrock port:

```bash
sudo ufw allow 25565/tcp comment 'Minecraft PaperMC'
sudo ufw allow 19132/udp comment 'Geyser Bedrock'
sudo ufw status verbose
```

---

## 9. Install Geyser for Bedrock Support

Geyser is a plugin that translates the Bedrock protocol so mobile and console players can connect to your Java server.

### 9.1 Download the Geyser plugin

```bash
sudo -u minecraft bash -c "
  curl -L -o /opt/minecraft/server/plugins/Geyser-Spigot.jar \
  https://download.geysermc.org/v2/projects/geyser/versions/latest/builds/latest/downloads/spigot
"
```

### 9.2 Restart to generate Geyser config

```bash
sudo systemctl restart minecraft
sudo journalctl -u minecraft -f
```

Wait for `Done (Xs)!`, then press `Ctrl+C`.

### 9.3 Set Geyser to offline auth mode

Because the server is running with `online-mode=false`, Geyser must also be set to offline mode. Otherwise Geyser will still attempt to verify Bedrock players against Microsoft's servers and reject them even though the server itself would accept them.

```bash
sudo -u minecraft nano /opt/minecraft/server/plugins/Geyser-Spigot/config.yml
```

Find this line:

```yaml
auth-type: online
```

Change it to:

```yaml
auth-type: offline
```

Save and restart:

```bash
sudo systemctl restart minecraft
```

### 9.4 Connect from a Bedrock client

On your mobile device or console, go to **Play > Servers > Add Server** and enter:

| Field | Value |
|-------|-------|
| Server Address | Your server's local IP (find it with `ip a`) |
| Port | 19132 |

---

## 10. Managing the Server

Common day-to-day commands:

```bash
# Start / Stop / Restart
sudo systemctl start minecraft
sudo systemctl stop minecraft
sudo systemctl restart minecraft

# View live logs
sudo journalctl -u minecraft -f

# View last 100 lines
sudo journalctl -u minecraft -n 100 --no-pager

# Check service status
sudo systemctl status minecraft
```

### Making yourself an operator

Edit `ops.json` directly before starting the server, or send the command via RCON if configured:

```bash
sudo -u minecraft nano /opt/minecraft/server/ops.json
```

Add your entry:

```json
[
  {
    "uuid": "your-minecraft-uuid",
    "name": "YourUsername",
    "level": 4,
    "bypassesPlayerLimit": false
  }
]
```

### Adding plugins

Drop `.jar` plugin files into `/opt/minecraft/server/plugins/` and restart the service. Browse plugins at [hangar.papermc.io](https://hangar.papermc.io).

### Backing up world data

Add the following paths to your backup solution (Restic or otherwise):

```
/opt/minecraft/server/world
/opt/minecraft/server/world_nether
/opt/minecraft/server/world_the_end
```

---

## 11. Updating PaperMC

Stop the server, download the new jar, and restart:

```bash
sudo systemctl stop minecraft

sudo -u minecraft bash -c "
  cd /opt/minecraft/server
  MC_VER='1.21.11'
  USER_AGENT='your-server-name/1.0 (your-contact-or-domain)'

  PAPER_URL=\$(curl -s -H \"User-Agent: \$USER_AGENT\" \
    \"https://fill.papermc.io/v3/projects/paper/versions/\${MC_VER}/builds\" | \
    jq -r 'first(.[] | select(.channel == \"STABLE\") | .downloads.\"server:default\".url)')

  curl -L -H \"User-Agent: \$USER_AGENT\" -o paper.jar \"\$PAPER_URL\"
  echo 'Update complete.'
"

sudo systemctl start minecraft
```

Always back up your world data before updating.

---

## Notes

- The `minecraft` user has no sudo access by design. Install any system packages as your own user before switching.
- `online-mode=false` disables server-side account verification only. The Minecraft app always requires a Microsoft login to launch regardless of this setting.
- Geyser `auth-type: offline` is required when `online-mode=false`. Without it, Geyser will still reject Bedrock players by attempting its own Microsoft verification.
- Keep the PaperMC API `User-Agent` header non-generic. Requests using default curl/wget user agents may be rejected or rate-limited by PaperMC's API.