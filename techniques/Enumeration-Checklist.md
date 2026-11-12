---
technique: "Enumeration Checklist (Linux PrivEsc)"
os: Linux
categorie: Enumeration
tags: [technique, enumeration, privesc, linux]
---

# Enumeration Checklist — Linux PrivEsc

> À dérouler dès qu'on obtient un shell limité. Dans l'ordre.

## 1. Qui suis-je
```bash
id
whoami
sudo -l                 # commandes autorisées en tant qu'un autre user (demande parfois un mdp)
groups
```

## 2. Tâches automatiques
```bash
cat /etc/crontab
ls -la /etc/cron.d/ ; cat /etc/cron.d/*
ls -la /etc/cron.hourly/ /etc/cron.daily/
cat /var/spool/cron/crontabs/* 2>/dev/null
ps aux | grep -i <user>          # process d'un autre user = script lancé ?
```

## 3. Fichiers mal configurés
```bash
# SUID
find / -perm -4000 -type f 2>/dev/null | grep -vE '^/snap|^/usr/bin|^/usr/lib|^/bin|^/usr/sbin|^/sbin'
# SGID
find / -perm -2000 -type f 2>/dev/null
# Capabilities
getcap -r / 2>/dev/null
# Fichiers writable par moi (hors /proc /sys /tmp)
find / -writable -type f 2>/dev/null | grep -vE '/proc|/sys|/tmp'
# Fichiers appartenant à l'user cible
find / -user <user> -type f 2>/dev/null | grep -vE '/proc|/sys'
```

## 4. Infos système
```bash
uname -a ; cat /etc/os-release       # version kernel → exploit kernel ?
cat /etc/passwd                      # users, writable ?
ls -la /home/*                       # homes des autres users
```

## 5. Outils d'énum auto
- LinEnum : `./LinEnum.sh`
- linpeas : `./linpeas.sh`
- pspy : surveiller les process/cron en temps réel (super utile pour voir un cron qui passe)

> ⚠️ `find` par nom de fichier (ex. chercher un flag) :
> `find / -name "*flag*" 2>/dev/null`
