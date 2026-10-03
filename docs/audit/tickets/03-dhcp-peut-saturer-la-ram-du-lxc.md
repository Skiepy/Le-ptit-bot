# [03] `--dhcp` peut saturer la RAM du LXC

**Priorité :** P0  
**Vérification :** ✅ Exécuté  
**Où :** `bot.py:3562-3622`

## Problème
`[str(ip) for ip in network]` matérialise toutes les adresses du réseau demandé, sans limite de taille.

## Preuve
Pic de RAM mesuré sur la vraie commande (attente de 45 s court-circuitée) :
- `10.0.0.0/10` (4 M d'adresses) : +290 Mo, boucle bloquée 4,7 s.
- `10.0.0.0/8` (16 M) : **+1,16 Go**, 19 s.
- `0.0.0.0/0` : 256 × `/8`, soit environ 300 Go (extrapolé, non testé). Le processus sera tué par l'OOM killer.

Pas de cooldown sur cette commande.

## Correctif proposé
- Refuser les réseaux plus grands qu'un `/22`, par exemple, via `network.num_addresses`.
- Ne générer que le nombre d'IP nécessaire (`itertools.islice(network.hosts(), len(users) + 1)`).
- Gérer le cas « plus de participants que d'IP » (`IndexError` actuel sur `ips.pop(0)`).
- Ajouter un cooldown.

## Critères de validation
- [ ] `--dhcp 0.0.0.0/0` est refusé immédiatement.
- [ ] Un `/24` fonctionne comme avant.

_Référence : `docs/audit/AUDIT.md`_
