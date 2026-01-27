# Telegram Bot Guide V2

This guide explains how to build and run a Telegram-based Linux server maintenance assistant. The bot helps you safely update and upgrade packages, monitor system health, and receive scheduled status reports — all from Telegram.

---

## What You'll Have When Done

By the end of this guide, you'll have a practical, no-nonsense maintenance system that feels like a personal SRE living inside your server:

---

## Reliable Package Maintenance

- On-demand apt update and apt upgrade via Telegram commands
- Clear, human-readable execution feedback
- Protection against overlapping runs (no double upgrades at 3 AM)
- Optional reboot awareness after kernel updates

---

## Personal Maintenance Bot

- Secure Telegram bot restricted to your chat ID
- Clean command set for maintenance tasks
- Descriptive bot profile (About, Description, and commands)
- Fast responses without flooding your chat

---

## Architecture Overview

The Wacky Maintenance Bot is intentionally simple and boring — because boring systems stay alive.

- `Telegram Bot` – Command interface
- `Shell` – Executes maintenance tasks
- `systemd Service` – Keeps the bot running

---

## Core Features

### Package Management Commands

- `/update` – Refresh package lists
- `/upgrade` – Upgrade installed packages

---

## Security Model

This bot is powerful, so security is non-negotiable:
- Chat ID allowlist (only you can talk to it)
- No inbound network ports exposed
- Sudo rules scoped to specific commands only
- Logs everything it runs

---

## Why This Bot Exists

Servers don’t need babysitters. They need consistent, repeatable maintenance.

This bot exists to:
- Reduce SSH logins
- Prevent forgotten updates
- Give fast visibility without dashboards
- Keep things running quietly in the background

---

### Step 1: Create the Telegram Bot

1. Message **@BotFather** on Telegram
2. click **/start**
3. Send **/newbot**
4. Give it a name like **"Maintenance Telegram Bot"**
5. Give it a username like **"your_maintenance_bot"**
6. Copy the bot token **(looks like 123456789:ABCdefGhIjKlMnOpQrStUvWxYz)**

---

### Step 2: Get your Chat ID and Test Bot

1. Message your new bot
2. Clic **/start**
3. Send a message
4. Visit: **https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates**
5. Find your chat_id in the JSON response, something like this:

```bash
"chat": {
          "id": ,
```

If this doesn't work, try this method I used:

```bash
curl https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates
```

---

### Step 3: Create another user 

Create a dedicated user:
```bash
sudo adduser tg-maint
```

Edit sudoers:
```bash
sudo nano-f /etc/sudoers.d/tg-maint
```

Paste:
```bash
tg-maint ALL=(root) NOPASSWD: /usr/bin/apt
tg-maint ALL=(root) NOPASSWD: /opt/tg-maint/tg-maint-bot.sh
```

---

### Step 4: Create the directory and file

Directory:
```bash
sudo mkdir -p /opt/tg-maint

cd /opt/tg-maint
```

File:
```bash
sudo touch tg_maint_offset

sudo chown tg-maint:tg-maint /opt/tg-maint/tg_maint_offset

sudo chmod 644 /opt/tg-maint/tg_maint_offset
```

---

### Step 5: Create the script

**Note:** Fill in the double quotation with your `Bot Token` and `Chat ID`.

Create the .sh file
```bash
sudo nano tg-maint-bot.sh
```

Paste this:
```bash
#!/bin/bash
# System Maintenance Executor - tg-maint
# No Telegram logic here

case "$1" in
    update)
        output=$(sudo /usr/bin/apt update -y -o Dpkg::Progress-Fancy=0 2>&1)
        echo "$output" | grep -v "WARNING: apt does not have a stable CLI interface"
        ;;
    upgrade)
        output=$(sudo /usr/bin/apt upgrade -y 2>&1)
        echo "$output" | grep -v "WARNING: apt does not have a stable CLI interface"
        ;;
    status)
        output=$(sudo /usr/bin/apt list --upgradable 2>&1)
        clean_output=$(echo "$output" | grep -v "WARNING: apt does not have a stable CLI interface")
        
        # Check if there's anything after "Listing..."
        packages=$(echo "$clean_output" | grep -v "^Listing\.\.\.$")
        
        if [ -z "$packages" ]; then
            echo "✓ System is up to date. No packages need upgrading."
        else
            echo "$clean_output"
        fi
        ;; 
    reboot)
        sudo /sbin/reboot
        ;;
    *)
        echo "Usage: $0 {update|upgrade|status|reboot}"
        exit 1
        ;;
esac
```

Save the script.

Change the owner and permissions:
```bash
sudo chown tg-maint:tg-maint /opt/tg-maint/tg-maint-bot.sh

sudo chmod 750 /opt/tg-maint/tg-maint-bot.sh
```

---

### Step 6: Configure System Service

Now create a systemd service to run it 24/7:
```bash
sudo nano /etc/systemd/system/tg-maint.service
```

Service config:
```bash
[Unit]
Description=Your Telegram Maintenance Bot
After=network.target

[Service]
Type=simple
User=tg-maint
Group=tg-maint
WorkingDirectory=/opt/tg-maint
ExecStart=/opt/tg-maint/bot_command_handler.sh listen
Restart=always
RestartSec=5
Environment=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

Start the monitoring:
```bash
sudo systemctl daemon-reload
sudo systemctl enable tg-maint
sudo systemctl start tg-maint
```

---

### Step 7: Add Interactive Commands

#### Step 1: Configure Bot Commands

1. Type **/mybots** to **BotFather**
2. Click on your bot
3. Then click **"Edit Bot"**
4. Then click **"Edit Commands"**
   Paste this:

```bash
start - Show maintenance menu
update - Run apt update
upgrade - Run apt upgrade
status - Show system status
reboot - Reboot server now
help - Help information
```

#### Step 2: Create Command Handler Script

**Note:** Fill in the double quotation with your `Bot Token` and `Chat ID`.

Change directory:
```bash
cd /opt/tg-maint
```

Create the script:
```bash
sudo nano bot_command_handler.sh
```

Paste the script:
```bash
#!/bin/bash
# Telegram Maintenance Bot - Command Listener Only
# Uses BotFather command menu (no inline buttons)

BOT_TOKEN="" # Get from @BotFather
AUTHORIZED_CHAT_ID="" # Get from /getUpdates
OFFSET_FILE="/opt/tg-maint/tg_maint_offset"
TG_MAINT="/opt/tg-maint/tg-maint-bot.sh"

sanitize_output() {
    sed 's/[^[:print:]\t]//g'
}

html_escape() {
    sed -e 's/&/\&amp;/g' \
        -e 's/</\&lt;/g' \
        -e 's/>/\&gt;/g'
}

send_message() {
    local chat_id="$1"
    local text="$2"
    local mode="${3:-}"

    curl -s -X POST "https://api.telegram.org/bot${BOT_TOKEN}/sendMessage" \
        --data-urlencode "chat_id=${chat_id}" \
        --data-urlencode "text=${text}" \
        ${mode:+--data-urlencode "parse_mode=${mode}"}
}

send_code_block() {
    local chat_id="$1"
    local text="$2"
    local max=3500

    text=$(echo "$text" | sanitize_output | html_escape)

    if [ -z "$text" ]; then
        send_message "$chat_id" "<i>Command completed with no output.</i>" HTML
        return
    fi

    text="${text:0:$max}"

    send_message "$chat_id" "<pre>$text</pre>" HTML
}

handle_start() {
    send_message "$1" \
"Welcome to *Your Maintenance Bot*

Available commands:
/update  - Run apt update
/upgrade - Run apt upgrade
/status  - Show upgradable packages
/reboot  - Reboot server now
/help    - Show this message"
}

handle_help() { handle_start "$1"; }

handle_update() {
    send_message "$AUTHORIZED_CHAT_ID" "Running apt update..."

    output=$(sudo "$TG_MAINT" update 2>&1)

    send_code_block "$AUTHORIZED_CHAT_ID" "$output"
}

handle_upgrade() {
    send_message "$AUTHORIZED_CHAT_ID" "Running apt upgrade..."

    output=$(sudo "$TG_MAINT" upgrade 2>&1)

    send_code_block "$AUTHORIZED_CHAT_ID" "$output"
}

handle_status() {
    output=$(sudo "$TG_MAINT" status 2>&1)
    send_code_block "$AUTHORIZED_CHAT_ID" "$output"
}

handle_reboot() {
    send_message "$AUTHORIZED_CHAT_ID" \
"*Confirm reboot*
Reply with:
/reboot_yes
/reboot_no" Markdown
}

check_messages() {
    local offset=0
    [ -f "$OFFSET_FILE" ] && offset=$(cat "$OFFSET_FILE")

    local updates
    updates=$(curl -s "https://api.telegram.org/bot${BOT_TOKEN}/getUpdates?offset=$((offset+1))&timeout=10")

    echo "$updates" | jq -r '.result[]? | @base64' | while read -r row; do
        [ -z "$row" ] && continue

        local decoded
        decoded=$(echo "$row" | base64 -d)

        local update_id
        update_id=$(echo "$decoded" | jq -r '.update_id')

        local chat_id
        chat_id=$(echo "$decoded" | jq -r '.message.chat.id // empty')

        local text
        text=$(echo "$decoded" | jq -r '.message.text // empty')

        echo "$update_id" > "$OFFSET_FILE"

        [ "$chat_id" != "$AUTHORIZED_CHAT_ID" ] && continue

        case "$text" in
            /start|/start@YourMaintenanceBot) handle_start "$chat_id" ;;
            /help|/help@YourMaintenanceBot) handle_help "$chat_id" ;;
            /update|/update@YourMaintenanceBot) handle_update ;;
            /upgrade|/upgrade@YourMaintenanceBot) handle_upgrade ;;
            /status|/status@YourMaintenanceBot) handle_status ;;
            /reboot|/reboot@YourMaintenanceBot) handle_reboot ;;
	    /reboot_yes|/reboot_yes@YourMaintenanceBot)
   		 send_message "$AUTHORIZED_CHAT_ID" "Rebooting now..."
   		 "$TG_MAINT" reboot
   		 ;;
	    /reboot_no|/reboot_no@YourMaintenanceBot)
   		 send_message "$AUTHORIZED_CHAT_ID" "Reboot cancelled."
  		 ;;

        esac
    done
}

case "$1" in
    listen)
        echo "Telegram maintenance bot listening..."
        while true; do
            check_messages
            sleep 2
        done
        ;;
    *)
        echo "Usage: $0 listen"
        ;;
esac
```

Save the script.

Change the owner and permission:
```bash
sudo chown tg-maint:tg-maint /opt/tg-maint/bot_command_handler.sh

sudo chmod 755 /opt/tg-maint/bot_command_handler.sh
```