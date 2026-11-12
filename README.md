# TryHackMe — Writeups

Writeups personnels de rooms TryHackMe, dans le cadre du parcours **Jr Penetration Tester**.

Chaque writeup suit la même structure : reconnaissance, accès initial, élévation de privilèges, et une section "ce que j'ai appris" avec les pièges rencontrés. Les flags sont volontairement masqués (`THM{REDACTED}`), conformément aux bonnes pratiques TryHackMe.

## Writeups

- [jump](writeups/jump.md) — chaîne de privilege escalation Linux à 5 paliers (recon -> dev -> monitor -> ops -> root) : pipeline FTP + cron, scripts group-writable, PATH hijacking, abus de sudo, GTFOBins.

## Approche

L'objectif de ces notes n'est pas de collectionner des flags, mais de construire une méthode réutilisable. Le fil conducteur d'une escalade de privilèges reste le même : repérer une action effectuée par un compte plus privilégié qui dépend d'un élément que l'on contrôle (un script modifiable, une commande appelée sans chemin absolu, une configuration sudo permissive).

## À propos

Dylan Mehadi — étudiant en BUT Réseaux & Télécommunications (parcours cybersécurité, labellisé SecNumedu), en alternance chez Orange.
