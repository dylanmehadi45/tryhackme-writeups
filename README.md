# TryHackMe — Writeups & Fiches techniques

Notes personnelles de cybersécurité offensive, dans le cadre du parcours **Jr Penetration Tester** (TryHackMe).

Deux parties : des **writeups** de rooms (démarche complète) et des **fiches techniques** réutilisables (une technique = un principe, les conditions d'exploitation, le diagnostic, la remédiation). Les flags sont volontairement masqués (`THM{REDACTED}`), conformément aux bonnes pratiques TryHackMe.

## Writeups

### Linux
- [jump](writeups/jump.md) — chaîne de privilege escalation Linux à 5 paliers (recon -> dev -> monitor -> ops -> root) : pipeline FTP + cron, scripts group-writable, PATH hijacking, abus de sudo, GTFOBins.

### Windows
- [Windows Jump](writeups/windows-jump.md) — chaîne de privilege escalation Windows à 4 paliers (thmuser -> notadmin -> svcadmin -> SYSTEM) : creds SMB dans un partage, autologon en clair (Winlogon), binaire de service écrasable, script de tâche legacy inscriptible (C:\Windows\Tasks) lancé en SYSTEM.

## Fiches techniques

### Transverse
- [Méthodologie pentest](techniques/Methodologie-Pentest.md)
- [Checklist d'énumération (Linux PrivEsc)](techniques/Enumeration-Checklist.md)

### Linux — élévation de privilèges
- [PATH hijacking](techniques/linux/PATH-Hijacking.md)
- [Binaires SUID](techniques/linux/SUID-Binaries.md)
- [Capabilities Linux](techniques/linux/Linux-Capabilities.md)
- [Tâches cron](techniques/linux/Cron-Jobs.md)
- [Exploitation de sudo](techniques/linux/Sudo-Exploitation.md)
- [/etc/passwd inscriptible](techniques/linux/Writable-etc-passwd.md)
- [Pipeline de traitement de fichiers (dépôt -> exécution)](techniques/linux/Service-Exploitation-Cron.md)

### Windows — élévation de privilèges
- [Théorie — Windows PrivEsc](techniques/windows/00-Theorie-Windows-PrivEsc.md)
- [Méthodologie — Windows PrivEsc (diagnostic par vecteur)](techniques/windows/00-Methodologie-Windows-PrivEsc.md)
- [Permissions faibles sur l'exécutable d'un service](techniques/windows/Service-Insecure-Permissions.md)
- [Chemin de service non entre guillemets](techniques/windows/Unquoted-Service-Path.md)
- [DACL de service faible](techniques/windows/Weak-Service-DACL.md)
- [SeImpersonate (Potato)](techniques/windows/SeImpersonate.md)
- [SeBackupPrivilege](techniques/windows/SeBackupPrivilege.md)
- [SeTakeOwnership](techniques/windows/SeTakeOwnership.md)

### Web
- [Command injection](techniques/web/Command-Injection.md)

## Approche

L'objectif de ces notes n'est pas de collectionner des flags, mais de construire une méthode réutilisable. Le fil conducteur d'une escalade de privilèges reste le même : repérer une action effectuée par un compte plus privilégié qui dépend d'un élément que l'on contrôle (un script modifiable, une commande appelée sans chemin absolu, une configuration sudo permissive).

## À propos

Dylan Mehadi — étudiant en BUT Réseaux & Télécommunications (parcours cybersécurité, labellisé SecNumedu), en alternance chez Orange.
