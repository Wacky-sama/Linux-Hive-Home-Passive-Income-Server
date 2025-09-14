# This file is just an example

```bash
sudo nano /etc/fail2ban/jail.local
```

Add this:
```bash
[DEFAULT]
# Ban for 1 hour after 3 failed attempts in 10 minutes
bantime = 3600
findtime = 600
maxretry = 3

# Email notifications (optional, leave blank if you don't want emails)
destemail = 
sender = 
action = %(action_)s

[sshd]
enabled = true
port = your_port
filter = sshd
backend = systemd
maxretry = 3
bantime = 3600
findtime = 600
```

Restart fail2ban:
```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

To test - try this from another machine:
Try to SSH with wrong password a few times
```bash
ssh -p your_port wronguser@your-server-ip
```

You should see the attempts in:
```bash
sudo journalctl -u ssh -f
```

And bans happening in:
```bash
sudo tail -f /var/log/fail2ban.log
```

Check banned IPs:
```bash
sudo fail2ban-client status sshd
sudo iptables -L f2b-sshd
```

To unban an IP:
```bash
sudo fail2ban-client set sshd unbanip ip_of_banned_user
```