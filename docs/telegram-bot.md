# Telegram Bot Guide

### Step 1: Create the Telegram Bot
1. Message @BotFather on Telegram
2. click /start
3. Send /newbot
4. Give it a name like "SSH Security Bot"
5. Give it a username like "your_ssh_security_bot"
6. Copy the bot token (looks like 123456789:ABCdefGhIjKlMnOpQrStUvWxYz)

### Step 2: Get your Chat ID
1. Message your new bot
2. Clic /start
3. Send a message
4. Visit: https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates
5. Find your chat_id in the JSON response, something like this:
```bash
"chat": {
          "id": 23131231245,
```

If this doesn't work, try this method I used:
```bash
curl https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates
```

Creating monitoring directory
```bash
sudo mkdir -p /opt/ssh-monitor
cd /opt/ssh-monitor
```

```bash
#!/bin/bash

# Configuration
BOT_TOKEN=""
CHAT_ID=""
HOSTNAME=$(hostname)
SERVER_IP=$(hostname -I | awk '{print $1}')

# Emojis for style 💯
EMOJI_BAN="🚫"
EMOJI_SUCCESS="✅" 
EMOJI_FAIL="❌"
EMOJI_UNBAN="🔓"
EMOJI_STATS="📊"
EMOJI_ALERT="🚨"
EMOJI_ROBOT="🤖"

# Function to send Telegram message
send_telegram() {
    local message="$1"
    curl -s -X POST \
        "https://api.telegram.org/bot${BOT_TOKEN}/sendMessage" \
        -d "chat_id=${CHAT_ID}" \
        -d "text=${message}" \
        -d "parse_mode=Markdown" >/dev/null 2>&1
}

# Function to get current timestamp
get_timestamp() {
    date '+%Y-%m-%d %H:%M:%S'
}

# Monitor SSH attempts
monitor_ssh() {
    local logfile="/tmp/ssh_monitor.log"
    
    # Initialize log position
    if [ ! -f "$logfile" ]; then
        journalctl -u ssh --since "1 minute ago" -n 0 > "$logfile"
    fi
    
    # Get new log entries
    local new_logs=$(journalctl -u ssh --since "1 minute ago" -o short-iso)
    
    # Process new entries
    while IFS= read -r line; do
        local timestamp=$(echo "$line" | awk '{print $1}')
        local message=$(echo "$line" | cut -d' ' -f4-)
        
        # Failed password attempts
        if echo "$line" | grep -q "Failed password"; then
            local user=$(echo "$line" | grep -o "Failed password for [^ ]*" | awk '{print $4}')
            local ip=$(echo "$line" | grep -o "from [0-9.]*" | awk '{print $2}')
            send_telegram "${EMOJI_FAIL} *SSH Failed Login*
${EMOJI_ROBOT} Server: \`${HOSTNAME}\`
👤 User: \`${user}\`
🌐 IP: \`${ip}\`
🕐 Time: \`$(get_timestamp)\`"
        fi
        
        # Successful logins
        if echo "$line" | grep -q "Accepted publickey"; then
            local user=$(echo "$line" | grep -o "Accepted publickey for [^ ]*" | awk '{print $4}')
            local ip=$(echo "$line" | grep -o "from [0-9.]*" | awk '{print $2}')
            send_telegram "${EMOJI_SUCCESS} *SSH Successful Login*
${EMOJI_ROBOT} Server: \`${HOSTNAME}\`
👤 User: \`${user}\`
🌐 IP: \`${ip}\`
🕐 Time: \`$(get_timestamp)\`"
        fi
        
        # Invalid user attempts
        if echo "$line" | grep -q "Invalid user"; then
            local user=$(echo "$line" | grep -o "Invalid user [^ ]*" | awk '{print $3}')
            local ip=$(echo "$line" | grep -o "from [0-9.]*" | awk '{print $2}')
            send_telegram "${EMOJI_ALERT} *SSH Invalid User*
${EMOJI_ROBOT} Server: \`${HOSTNAME}\`
👤 Attempted User: \`${user}\`
🌐 IP: \`${ip}\`
🕐 Time: \`$(get_timestamp)\`"
        fi
        
    done <<< "$new_logs"
}

# Monitor fail2ban
monitor_fail2ban() {
    local logfile="/var/log/fail2ban.log"
    local position_file="/tmp/fail2ban_position"
    
    # Initialize position file
    if [ ! -f "$position_file" ]; then
        wc -l < "$logfile" > "$position_file"
        return
    fi
    
    local last_line=$(cat "$position_file")
    local current_line=$(wc -l < "$logfile")
    
    if [ "$current_line" -gt "$last_line" ]; then
        local new_lines=$((current_line - last_line))
        tail -n "$new_lines" "$logfile" | while IFS= read -r line; do
            # Ban notifications
            if echo "$line" | grep -q "NOTICE.*Ban"; then
                local ip=$(echo "$line" | grep -o "Ban [0-9.]*" | awk '{print $2}')
                local jail=$(echo "$line" | grep -o "\[.*\]" | tr -d '[]')
                send_telegram "${EMOJI_BAN} *IP BANNED!*
${EMOJI_ROBOT} Server: \`${HOSTNAME}\`
🚫 Banned IP: \`${ip}\`
🔒 Jail: \`${jail}\`
🕐 Time: \`$(get_timestamp)\`
💪 *Get rekt, script kiddie!*"
            fi
            
            # Unban notifications
            if echo "$line" | grep -q "NOTICE.*Unban"; then
                local ip=$(echo "$line" | grep -o "Unban [0-9.]*" | awk '{print $2}')
                local jail=$(echo "$line" | grep -o "\[.*\]" | tr -d '[]')
                send_telegram "${EMOJI_UNBAN} *IP Unbanned*
${EMOJI_ROBOT} Server: \`${HOSTNAME}\`
🔓 Unbanned IP: \`${ip}\`
🔒 Jail: \`${jail}\`
🕐 Time: \`$(get_timestamp)\`"
            fi
        done
        
        echo "$current_line" > "$position_file"
    fi
}

# Generate daily stats
daily_stats() {
    local today=$(date '+%Y-%m-%d')
    local banned_today=$(fail2ban-client status sshd | grep "Total banned" | awk '{print $4}')
    local failed_today=$(journalctl -u ssh --since "today" | grep -c "Failed password" || echo "0")
    local success_today=$(journalctl -u ssh --since "today" | grep -c "Accepted publickey" || echo "0")
    local invalid_today=$(journalctl -u ssh --since "today" | grep -c "Invalid user" || echo "0")
    
    send_telegram "${EMOJI_STATS} *Daily SSH Stats - ${today}*
${EMOJI_ROBOT} Server: \`${HOSTNAME}\` (${SERVER_IP})

${EMOJI_SUCCESS} Successful logins: \`${success_today}\`
${EMOJI_FAIL} Failed attempts: \`${failed_today}\`
${EMOJI_ALERT} Invalid users: \`${invalid_today}\`
${EMOJI_BAN} Total bans: \`${banned_today}\`

🛡️ *Your fortress is secure!*"
}

# Main monitoring loop
case "${1}" in
    "monitor")
        while true; do
            monitor_ssh
            monitor_fail2ban
            sleep 60
        done
        ;;
    "stats")
        daily_stats
        ;;
    "test")
        send_telegram "${EMOJI_ROBOT} *SSH Monitor Test*
Server: \`${HOSTNAME}\`
Status: \`Online and monitoring\`
Time: \`$(get_timestamp)\`
🔥 *Ready to catch hackers!*"
        ;;
    *)
        echo "Usage: $0 {monitor|stats|test}"
        echo "  monitor - Start continuous monitoring"
        echo "  stats   - Send daily statistics"
        echo "  test    - Send test message"
        exit 1
        ;;
esac
```

Save the script
```bash
sudo nano ssh_telegram_monitor.sh
```

Make it executable
```bash
sudo chmod +x ssh_telegram_monitor.sh
```

Test it works
```bash
sudo ./ssh_telegram_monitor.sh test
```
Check your phone - you should get a test message! 

Now create a systemd service to run it 24/7:
```bash
sudo nano /etc/systemd/system/ssh-monitor.service
```
Service config:
```bash
[Unit]
Description=SSH Telegram Monitor
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/ssh-monitor
ExecStart=/opt/ssh-monitor/ssh_telegram_monitor.sh monitor
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Start the monitoring:
```bash
sudo systemctl daemon-reload
sudo systemctl enable ssh-monitor
sudo systemctl start ssh-monitor
```

Crontab = Cron Table - it's Linux's built-in scheduler! It runs commands automatically at specific times.
Set up daily stats (crontab):
```bash
sudo crontab -e
```
Add this line:
```bash
0 9 * * * /opt/ssh-monitor/ssh_telegram_monitor.sh stats
```
It is like this:
```bash
minute hour day month weekday command
  |     |    |    |      |       |
  |     |    |    |      |       +-- Command to run
  |     |    |    |      +---------- Day of week (0-7, Sunday=0 or 7)  
  |     |    |    +----------------- Month (1-12)
  |     |    +---------------------- Day of month (1-31)
  |     +--------------------------- Hour (0-23)
  +--------------------------------- Minute (0-59)
```