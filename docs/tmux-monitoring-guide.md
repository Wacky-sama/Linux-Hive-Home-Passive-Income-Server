# Tmux Monitoring Guide

This guide will show how to setup tmux auto-resurrect upon reboot of the server

## Step 1: Create the directory for the user script

```bash
mkdir -p ~/.local/bin
```

## Step 2: Create the startup script

Use your preferred text editor(`Nano` or `Vim`)

```bash
vim ~/.local/bin/tmux-monitor.sh
```

Paste this:

```bash
#!/bin/bash

SESSION="monitor"

# Exit if the session already exists
tmux has-session -t "$SESSION" 2>/dev/null && exit 0

# Create the session
tmux new-session -d -s "$SESSION"

# First pane
P0=$(tmux display-message -p -t "$SESSION":0.0 "#{pane_id}")
tmux send-keys -t "$P0" "journalctl -u tg-maint.service -f" C-m

# Right pane
P1=$(tmux split-window -h -P -F "#{pane_id}" -t "$P0")
tmux send-keys -t "$P1" "journalctl -u daily-commit.service -f" C-m

# Bottom-right
P2=$(tmux split-window -v -P -F "#{pane_id}" -t "$P1")
tmux send-keys -t "$P2" "docker logs -tf honeygain" C-m

# Bottom-left
P3=$(tmux split-window -v -P -F "#{pane_id}" -t "$P0")
tmux send-keys -t "$P3" "journalctl -u pawns.service -f" C-m

# Bottom under Docker
P4=$(tmux split-window -v -P -F "#{pane_id}" -t "$P2")
tmux send-keys -t "$P4" "journalctl -u server-backup.service -f" C-m

tmux select-layout -t "$SESSION":0 tiled
```

## Step 3. Make it executable

```bash
chmod +x ~/.local/bin/tmux-monitor.sh
```

## Step 4. Test it manually

Run:

```bash
~/.local/bin/tmux-monitor.sh
```

Check:

```bash
tmux ls
```

You should see

```bash
monitor: 1 windows
```

To attach:

```bash
tmux attach -t monitor
```

## Step 5. Create a user systemd service

Create the directory if needed:

```bash
mkdir -p ~/.config/systemd/user
```

Create the service:

```bash
vim ~/.config/systemd/user/tmux-monitor.service
```

Paste:

```bash
[Unit]
Description=Start tmux monitoring session

[Service]
Type=oneshot
ExecStart=%h/.local/bin/tmux-monitor.sh
RemainAfterExit=yes

[Install]
WantedBy=default.target
```

## Step 6. Reload systemd

```bash
systemctl --user daemon-reload
```

## Step 7. Enable and Start it

```bash
systemctl --user enable tmux-monitor.service
systemctl --user start tmux-monitor.service
```

Check the status with:

```bash
systemctl --user status tmux-monitor.service
```

## Step 8. Make it survive reboots without logging in

Run `once`:

```bash
sudo loginctl enable-linger $USER
```

This tells systemd to keep your user instance running even when you're not logged in, so your tmux session starts automatically at boot.

## Step 9. Reboot and verify

After rebooting:

```bash
tmux ls
```

You should see:

```bash
monitor: 1 windows
```