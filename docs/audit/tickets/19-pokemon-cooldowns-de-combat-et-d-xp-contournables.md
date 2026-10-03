# [19] Pokémon : cooldowns de combat et d'XP contournables

**Priorité :** P2  
**Vérification :** 📖 Lecture du code  
**Où :** `cogs/pokemon_starter.py:1158-1189` (combat), `:741-756` (XP)

## Problème
- **Combat** : `last_combat` n'est écrit qu'après l'acceptation (jusqu'à 60 s plus tard), et seul le cooldown du lanceur est vérifié. On peut donc lancer plusieurs défis en parallèle, et défier quelqu'un qui est lui-même en cooldown.
- **XP** : `entry["last_time"]` n'est mis à jour qu'après `evolve()`, qui fait plusieurs `sleep` (~4 s). Pendant l'animation, chaque message du joueur rapporte de l'XP.

## Preuve
Déduit de la lecture du code.

## Correctif proposé
- Combat : vérifier le cooldown des deux joueurs, et poser `last_combat` (ou un verrou « en combat ») dès l'envoi du défi.
- XP : écrire `entry["last_time"] = now` avant le moindre `await`.

## Critères de validation
- [ ] Deux `/combat` lancés coup sur coup : le second est refusé.

_Référence : `docs/audit/AUDIT.md`_
