---
technique: "PATH Hijacking"
os: Linux
categorie: PrivEsc
tags: [technique, privesc, linux]
rooms: jump
---

# PATH Hijacking

## Principe
> Un programme privilégié appelle une commande **par son nom** (ex. `ps`, `tar`) sans chemin absolu. Le shell la cherche dans les dossiers de `$PATH`, de gauche à droite, et exécute **le premier trouvé**. Si je place mon faux binaire dans un dossier prioritaire, c'est le mien qui s'exécute — avec les droits du programme.

## Les 3 conditions pour que ça marche
1. Un programme **privilégié** (SUID, sudo, cron, service d'un autre user) appelle une commande **sans `/`** devant.
2. Je peux influencer le **`$PATH`** utilisé par ce programme (ou il définit un PATH qui inclut un dossier que je contrôle).
3. Je peux **écrire** mon faux binaire dans un dossier placé **avant** le vrai dans ce PATH.

## Détection / énumération
```bash
# Scripts / binaires lancés par un autre user
sudo -l
find / -perm -4000 -type f 2>/dev/null | grep -vE '^/snap|^/usr/bin|^/usr/lib|^/bin|^/usr/sbin|^/sbin'
# Lire un script cible : repérer une commande appelée sans chemin absolu
cat /chemin/vers/script
# Voir le PATH et un dossier écrivable en tête
echo $PATH
ls -la /opt/... /usr/local/bin
```

## Exploitation (schéma générique)
```bash
# 1. Créer un faux binaire du nom de la commande nue (ex. "ps") dans un dossier écrivable
#    contenu : la charge voulue (reverse shell vers MON IP, ou copie de bash, etc.)
# 2. Le rendre exécutable : chmod +x
# 3. S'assurer que ce dossier passe AVANT le vrai dans le PATH du programme
#    (export PATH=/mon/dossier:$PATH, ou déposer dans un dossier déjà prioritaire)
# 4. Déclencher le programme / attendre le cron/service
# 5. Récupérer le shell avec les droits du programme
```

## Remédiation (côté défense)
- Toujours appeler les commandes avec leur **chemin absolu** (`/usr/bin/ps`).
- Définir un `PATH` sûr en début de script (`PATH=/usr/sbin:/usr/bin:/sbin:/bin`).
- Permissions strictes sur les dossiers du PATH (pas de world-writable).

## Rooms où je l'ai vue
- jump — service `healthcheck` qui lance `ps` sans chemin absolu (recon_user → dev_user)
