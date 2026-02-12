# Nextcloud Update Guide

This guide will walk you through on how to update your Nextcloud using CLI.

---

## Step 1: Change directory to `/var/www/nextcloud`

```bash
cd /var/www/nextcloud
```

---

## Step 2: Turn on maintenance mode 

```bash
sudo -u www-data php occ maintenance:mode --on
```

---

## Step 3: Run the updater

```bash
sudo -u www-data php updater/updater.phar
```

- Start update? [y/N] y
- Should the "occ upgrade" command be executed? [Y/n] Y
- Keep maintenance mode active? [y/N] N

---

Done! Updated Nextcloud!