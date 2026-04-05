# Automated Backup with Restic + Systemd

**Server:** Wacky's Debian Hive (`debian`)
**Stack:** Restic, systemd timers, MariaDB dumps, Telegram notifications
**Repo ID:** `f39af59cd1` at `/srv/backups/restic-repo`

---

## Overview

This setup uses **Restic** as the backup engine with **systemd timers** (not cron) to schedule daily snapshots. A single daily run handles daily, weekly, and monthly retention automatically via Restic's `forget` policy. Telegram notifications are integrated into the existing `tg-maint` bot via a `/backup` command.

**Disk layout backed up:**

| Mount | Device | Size | Notes |
|---|---|---|---|
| `/etc` | sda4 | — | System config |
| `/home` | sda6 | 465.7G | User data |
| `/var` | sda5 | 139.7G | App data, Docker volumes |
| `/srv` | sda7 | 270.7G | Served content, backup repo lives here |
| `/mnt/msata` | sdb1 | 29.5G | MariaDB data dir |

**Excluded:** `/var/cache`, `/var/tmp`, `/srv/backups/restic-repo` (no recursive backup)

---

## Prerequisites

```bash
sudo apt install restic -y
```

Verify install:

```bash
restic version
# restic 0.18.0
```

---

## Step 1 — Initialize the Restic Repository

```bash
sudo restic init --repo /srv/backups/restic-repo
# enter password for new repository:
# enter password again:
# created restic repository f39af59cd1 at /srv/backups/restic-repo
```

> **Store this password somewhere safe.** Losing it = unrecoverable backups. No exceptions.

---

## Step 2 — Create the Backup Script

```bash
sudo vi /usr/local/bin/server-backup.sh
```

```bash
#!/bin/bash
# ============================================
# server-backup.sh — Wacky's Hive Backup
# ============================================

export RESTIC_REPOSITORY="/srv/backups/restic-repo"
export RESTIC_PASSWORD="YOUR_STRONG_PASSWORD_HERE"

BOT_TOKEN="YOUR_BOT_TOKEN"
CHAT_ID="YOUR_CHAT_ID"
LOG_FILE="/var/log/server-backup.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')
HOSTNAME=$(hostname)

tg_send() {
    curl -s -X POST "https://api.telegram.org/bot${BOT_TOKEN}/sendMessage" \
        --data-urlencode "chat_id=${CHAT_ID}" \
        --data-urlencode "text=${1}" \
        --data-urlencode "parse_mode=HTML" \
        > /dev/null
}

echo "[$DATE] ========== Backup started ==========" >> "$LOG_FILE"
tg_send "<b>Backup started</b> on <code>${HOSTNAME}</code> — ${DATE}"

# --- MariaDB Dump ---
MYSQL_DUMP_DIR="/srv/backups/mysql-dumps"
mkdir -p "$MYSQL_DUMP_DIR"

if mysqldump -u root --all-databases \
    --single-transaction \
    --routines \
    --triggers \
    > "$MYSQL_DUMP_DIR/all-databases-$(date +%F).sql" 2>> "$LOG_FILE"; then
    echo "[$DATE] MariaDB dump: OK" >> "$LOG_FILE"
else
    echo "[$DATE] MariaDB dump: FAILED" >> "$LOG_FILE"
    tg_send "<b>MariaDB dump FAILED</b> on <code>${HOSTNAME}</code>. Check logs."
    exit 1
fi

# --- Restic Snapshot ---
if restic backup \
    /etc \
    /home \
    /var \
    /srv \
    /mnt/msata \
    --exclude="/var/cache" \
    --exclude="/var/tmp" \
    --exclude="/srv/backups/restic-repo" \
    --tag "scheduled" \
    >> "$LOG_FILE" 2>&1; then
    echo "[$DATE] Snapshot: OK" >> "$LOG_FILE"
else
    echo "[$DATE] Snapshot: FAILED" >> "$LOG_FILE"
    tg_send "<b>Restic snapshot FAILED</b> on <code>${HOSTNAME}</code>. Check logs."
    exit 1
fi

# --- Forget & Prune (handles daily/weekly/monthly retention) ---
restic forget \
    --keep-daily 7 \
    --keep-weekly 4 \
    --keep-monthly 12 \
    --prune \
    >> "$LOG_FILE" 2>&1

SNAPSHOT_COUNT=$(restic snapshots --json 2>/dev/null | jq 'length')
END_DATE=$(date '+%Y-%m-%d %H:%M:%S')

echo "[$DATE] Backup finished successfully." >> "$LOG_FILE"
tg_send "<b>Backup complete</b> on <code>${HOSTNAME}</code>
${END_DATE}
Total snapshots: ${SNAPSHOT_COUNT}"
```

```bash
sudo chmod +x /usr/local/bin/server-backup.sh
```

> The `--keep-daily 7 --keep-weekly 4 --keep-monthly 12` policy means one daily run handles **all** retention tiers automatically. No separate weekly/monthly scripts needed.

---

## Step 3 — Create the Systemd Service

```bash
sudo vi /etc/systemd/system/server-backup.service
```

```ini
[Unit]
Description=Wacky Hive - Restic Backup
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/server-backup.sh
User=root
StandardOutput=journal
StandardError=journal
```

---

## Step 4 — Create the Systemd Timer

```bash
sudo vi /etc/systemd/system/server-backup.timer
```

```ini
[Unit]
Description=Wacky Hive - Daily Backup Timer

[Timer]
OnCalendar=*-*-* 03:00:00
AccuracySec=1min
Persistent=true

[Install]
WantedBy=timers.target
```

> `Persistent=true` — if the server was off at 3AM, it runs the backup on next boot instead of skipping it silently. This is why systemd timers beat cron for servers.

---

## Step 5 — Enable the Timer

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now server-backup.timer
```

Verify it's active:

```bash
systemctl list-timers --all | grep backup
```

---

## Step 6 — Integrate with tg-maint Bot

### `tg-maint-bot.sh` — add the `backup` case (must be ABOVE `*)`):

```bash
    backup)
        sudo /usr/local/bin/server-backup.sh
        ;;
    *)
        echo "Usage: $0 {update|upgrade|status|reboot|backup}"
        exit 1
        ;;
```

### `bot_command_handler.sh` — add handler function:

```bash
handle_backup() {
    send_message "$AUTHORIZED_CHAT_ID" "<b>Manual backup triggered.</b> You'll get a notification when it's done." HTML
    sudo "$TG_MAINT" backup &
}
```

### Add to the `case` block in `check_messages()`:

```bash
/backup|/backup@WackyMaintenanceBot)
    echo "$(date): /backup from $chat_id"
    handle_backup
    ;;
```

### Update the help menu in `handle_start()`:

```
/backup - Trigger a manual backup
```

### Update sudoers for `tg-maint`:

The bot runs as `tg-maint` user, so it needs explicit permission to call `server-backup.sh`:

```bash
sudo visudo -f /etc/sudoers.d/tg-maint
```

Add this line:

```
tg-maint ALL=(root) NOPASSWD: /usr/local/bin/server-backup.sh
```

Full `/etc/sudoers.d/tg-maint` should look like:

```bash
tg-maint ALL=(root) NOPASSWD: /usr/bin/apt
tg-maint ALL=(root) NOPASSWD: /sbin/reboot
tg-maint ALL=(root) NOPASSWD: /opt/tg-maint/tg-maint-bot.sh
tg-maint ALL=(root) NOPASSWD: /usr/local/bin/server-backup.sh
```

### Restart the bot service:

**Always restart tg-maint after editing any bot script** — the service holds the old version in memory until restarted:

```bash
sudo systemctl restart tg-maint.service
```

Verify it's back up:

```bash
systemctl status tg-maint.service
```

### Register with BotFather:

Send `/setcommands` to BotFather and add:

```
backup - Trigger a manual backup
```

---

## Useful Commands

```bash
# List all snapshots
sudo restic -r /srv/backups/restic-repo snapshots

# Verify backup integrity (run occasionally)
sudo restic -r /srv/backups/restic-repo check

# Restore a specific path from latest snapshot
sudo restic -r /srv/backups/restic-repo restore latest \
  --target /tmp/restore-test \
  --include /etc/apache2

# Check backup repo size
sudo restic -r /srv/backups/restic-repo stats

# View systemd journal for backup service
journalctl -u server-backup.service -f

# View flat log
tail -f /var/log/server-backup.log

# Check timer status
systemctl status server-backup.timer

# Manually trigger a backup run
sudo systemctl start server-backup.service
```

---

## Retention Policy Reference

| Flag | Keeps |
|---|---|
| `--keep-daily 7` | One snapshot per day for the last 7 days |
| `--keep-weekly 4` | One snapshot per week for the last 4 weeks |
| `--keep-monthly 12` | One snapshot per month for the last 12 months |

Restic automatically classifies each snapshot into the appropriate tier. No separate weekly/monthly cron jobs needed.

---

## Troubleshooting

**Service fails immediately:**
```bash
sudo bash -x /usr/local/bin/server-backup.sh
```
This traces every line. Common causes: wrong `RESTIC_PASSWORD`, repo not initialized, `/srv/backups` permission issue.

**Check permissions:**
```bash
sudo ls -la /srv/backups
# Should be: drwx------ root root
```

**Fix permissions if needed:**
```bash
sudo chown -R root:root /srv/backups
sudo chmod -R 755 /srv/backups
```

**Restic still running (first backup takes a while):**
```bash
sudo ps aux | grep restic
```
First snapshot is slow — subsequent runs are fast due to deduplication.