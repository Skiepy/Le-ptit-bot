# [09] Écritures de fichiers non atomiques : un crash peut effacer les données Pokémon

**Priorité :** P1  
**Vérification :** 📖 Lecture du code  
**Où :** `cogs/pokemon_starter.py:165-197`, `bot.py:405`, `bot.py:3270`

## Problème
`save_pokemon_data` réécrit tout `data/pokemon_starters.json` directement (`open(..., "w")`). Si le processus est tué pendant l'écriture (OOM dans le LXC, redémarrage…), le JSON est tronqué. Au démarrage suivant, `load_pokemon_data` attrape `JSONDecodeError` et **renvoie `{}`**. Le premier message reçu sauvegarde alors ce `{}` : toutes les données Pokémon sont perdues sans alerte.

Même schéma pour `dico.txt` (réécrit à chaque mot nouveau), `leaderboard.txt`, `pve.txt` et `server_names.txt`.

C'est le lien direct avec le risque LXC : les tickets mémoire (dhcp, prime, calcul) peuvent provoquer cette perte.

## Preuve
Déduit de la lecture du code, non reproduit.

## Correctif proposé
- Écriture atomique : écrire dans `fichier.tmp`, puis `os.replace(tmp, fichier)`.
- Sur `JSONDecodeError`, ne **pas** repartir de `{}` : renommer le fichier corrompu en `.corrupt-<date>` et refuser de sauvegarder.
- Ajouter une sauvegarde quotidienne de `data/`.

## Critères de validation
- [ ] Tuer le processus pendant une sauvegarde ne laisse jamais de JSON tronqué.

_Référence : `docs/audit/AUDIT.md`_
