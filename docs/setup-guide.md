# Server Setup Guide

## Table of Contents
1. [Installer Menu](#1-installer-menu)  
2. [Select Language](#2-select-language)  
3. [Select Location](#3-select-location)  
4. [Configure Keyboard](#4-configure-keyboard)  
5. [Set Hostname](#5-set-hostname)  
6. [Set Domain Name](#6-set-domain-name)  
7. [Root Password](#7-root-password)  
8. [Full Name](#8-full-name)  
9. [Username](#9-username)  
10. [User Password](#10-user-password)  
11. [Partition Disks](#11-partition-disks)  
12. [Partition Layout](#12-partition-layout)  
13. [Confirm Partitions](#13-confirm-partitions)  
14. [Mirror Country](#14-mirror-country)  
15. [Archive Mirror](#15-archive-mirror)  
16. [HTTP Proxy](#16-http-proxy)  
17. [Survey](#17-survey)  
18. [Software Selection](#18-software-selection)  
19. [Installation Complete](#19-installation-complete)  
20. [Add User to Sudoers](#add-user-to-sudoers)
    - [20.1 Switch to Root](#201-switch-to-root)  
    - [20.2 Add User to Sudo Group](#202-add-user-to-sudo-group)  
    - [20.3 Apply Changes](#203-apply-changes)  
---

## Installation Steps

### 1. Installer Menu
![Installer Menu](./images/Installer_Menu.png)  
*Choose "Install" or "Graphical Install" to begin the Debian setup.*

### 2. Select Language
![Select Language](./images/Select_Language.png)  
*Pick the language for your installation process and system.*

### 3. Select Location
![Select Location](./images/Select_Location.png)  
*Choose your country/region. This also affects time zones and mirrors.*

### 4. Configure Keyboard
![Configure Keyboard](./images/Config_Keyboard.png)  
*Select the keyboard layout you’ll use.*

### 5. Set Hostname
![Hostname](./images/Hostname.png)  
*Give your server a unique hostname (ex: `linux-hive`).*

### 6. Set Domain Name
![Domain Name](./images/Domain_Name.png)  
*Optional for home setups — you can leave it blank if not using domains.*

### 7. Root Password
![Root Password](./images/Root_Password.png)  
*Set a strong root password. Don’t lose this!*

### 8. Full Name
![Full Name](./images/Full_Name.png)  
*Enter the full name of the primary user (for reference only).*

### 9. Username
![Username](./images/Username.png)  
*Pick your login username (ex: `han`).*

### 10. User Password
![Password](./images/Password.png)  
*Set a strong password for your user account.*

### 11. Partition Disks
![Partition Disks](./images/Partition_Disks.png)  
*Select the method for partitioning. I used "Manual" for full control.*

### 12. Partition Layout
![Partition Layout](./images/Partition_Layout.png)  
*Review and adjust your partitions (/, /home, /srv, swap, etc.).*

### 13. Confirm Partitions
![Partition Disks 2](./images/Partition_Disks_2.png)  
*Write changes to disk to finalize the partitioning.*

### 14. Mirror Country
![Mirror Country](./images/Mirror_Country.png)  
*Select the country for your Debian package mirror.*

### 15. Archive Mirror
![Archive Mirror](./images/Archive_Mirror.png)  
*Pick the Debian archive mirror for downloading updates/packages.*

### 16. HTTP Proxy
![HTTP Proxy](./images/HTTP_Proxy.png)  
*Set this only if you’re behind a proxy. Otherwise, leave blank.*

### 17. Survey
![Survey](./images/Survey.png)  
*Debian asks if you want to participate in usage surveys. Choose yes/no.*

### 18. Software Selection
![Software Selection](./images/Software_Selection.png)  
*Choose which software to install. For servers, select SSH + standard utilities. If you also want to host a Web, choose Web Server*

### 19. Installation Complete
![Installation Complete](./images/Installation_Complete.png)  
*Remove installation media, reboot, and enjoy your fresh Debian server!*

## Add User to Sudoers
### 20.1 Switch to Root:
```bash
su
```
*(Enter the root password you set during installation.)*

### 20.2 Add User to Sudo Group:
```bash
/usr/sbin/usermod -aG sudo your_username
```
*(Replace your_username with your actual login username.)*

### 20.3 Apply Changes
Your user can now run admin commands with:
```bash
sudo <command>
```