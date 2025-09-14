# This file is just an example

## Edit sshd_config
```bash
sudo nano /etc/ssh/sshd_config
```
Follow these below or change it whatever you want.
```bash
Port 22 # or whatever random port you vibe with

# Authentication:

LoginGraceTime 30s
PermitRootLogin no
#StrictModes yes
MaxAuthTries 3
MaxSessions 2
AllowUsers your_username
ClientAliveInterval 300
ClientAliveCountMax 2
LoginGraceTime 30s

PubkeyAuthentication yes

PasswordAuthentication no
```
Before you restart SSH, do this safety check:
Test the config for syntax errors
```bash
sudo sshd -t
```
If that comes back clean, then:
```bash
sudo systemctl restart ssh
```