# Pawns.app Setup Guide

Pawns.app is a passive income app that shares your unused bandwidth in exchange for earnings. This guide covers installation and configuration on a Debian Linux server via SSH.

**Referral link:** https://pawns.app/?r=19512499

---

## Prerequisites

- Debian Linux (x86_64)
- SSH access to your server
- A registered Pawns.app account

---

## 1. Download the Binary

Pawns.app does not provide a public direct download link. You must retrieve it from your dashboard.

1. Log into [pawns.app](https://pawns.app/?r=19512499) on your local machine
2. Navigate to your dashboard and find the Linux download section
3. Right-click the **Linux x86 64** option → **Copy link address**
4. On your server, download it:

```bash
sudo mkdir -p /opt/pawns && cd /opt/pawns
sudo wget "https://paste-real-url-here" -O pawns-cli
sudo chmod +x pawns-cli
```

---

## 2. Test the Binary

Before setting up as a service, verify it works:

```bash
./pawns-cli -email=your@email.com -password=yourpassword -device-name=your-device-name -accept-tos
```

If it connects and shows activity, kill it with `Ctrl+C` and proceed.

---

## 3. Store Credentials Securely

Passing credentials directly as CLI flags exposes them in `systemctl status` output. Use an environment file instead.

```bash
sudo nano /etc/pawns.env
```

```ini
PAWNS_EMAIL=your@email.com
PAWNS_PASSWORD=yourpassword
```

```bash
sudo chmod 600 /etc/pawns.env
```

---

## 4. Create the systemd Service

```bash
sudo nano /etc/systemd/system/pawns.service
```

```ini
[Unit]
Description=Pawns.app CLI
After=network-online.target
Wants=network-online.target

[Service]
EnvironmentFile=/etc/pawns.env
Environment=HOME=/root
Type=simple
ExecStart=/opt/pawns/pawns-cli -email=${PAWNS_EMAIL} -password=${PAWNS_PASSWORD} -device-name=your-device-name -accept-tos
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

> `Restart=always` ensures the service recovers automatically after router reboots or connection drops, not just on failure exit codes.

---

## 5. Enable and Start

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now pawns
sudo systemctl status pawns
```

---

## 6. Verify It's Earning

```bash
journalctl -u pawns.service -f
```

Expected output when running correctly:

```json
{"name":"running","parameters":{}}
{"name":"balance_ready","parameters":{"balance":"0.200 USD","traffic":"1.0000 GB"}}
```

---

## Notes

- **Architecture:** This guide uses the `Linux x86 64` binary. If running on ARM hardware (e.g. Raspberry Pi), select the appropriate binary from the dashboard instead.
- **Network manager:** This setup assumes `NetworkManager` is active. Verify with `systemctl is-active NetworkManager`. If using `systemd-networkd` instead, enable `systemd-networkd-wait-online.service`.
- **Coexistence:** Pawns.app runs fine alongside other bandwidth-sharing apps such as Honeygain. They operate on separate networks and do not interfere with each other.
- **Router reboots:** The `After=network-online.target` directive combined with `Restart=always` handles reconnection gracefully after router reboots.