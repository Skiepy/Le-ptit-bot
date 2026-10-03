# [24] Hygiène du dépôt (à traiter sur le dépôt original)

**Priorité :** P2  
**Vérification :** ✅ Exécuté  
**Où :** Dépôt `NozyZy/Le-ptit-bot`

## Problème
- `txt/primes.txt` fait 97 Mo, proche de la limite GitHub de 100 Mo. 7 versions de 83 à 97 Mo sont dans l'historique (pack de 67 Mo). Inutile, cf. ticket `--prime`.
- Les fichiers d'état modifiés en prod sont versionnés : `tg.txt`, `onecops_counter.txt`, `leaderboard.txt`, `pve.txt`, `bans.txt`, `mature.txt`, `dico.txt`, `server_names.txt`, mèmes générés.
- 3174 fichiers d'un `venv/` ont été commités par le passé.
- `.gitignore` de 961 lignes (template générique).
- Bits exécutables sur des images, le README, la licence (déjà corrigé sur la branche `claude/cleanup-security-review` du fork).

## Preuve
Constaté avec `git rev-list --objects`, `git count-objects` et `git ls-files -s`.

## Correctif proposé
- Déplacer l'état dans `data/` (déjà ignoré), et fournir des fichiers `*.example` pour l'initialisation.
- `git rm --cached` des fichiers d'état et de `primes.txt`.
- Ne pas réécrire l'historique (ça casserait les forks), sauf décision explicite du propriétaire.
- **À faire upstream** : si on le fait seulement dans le fork, chaque synchronisation créera des conflits.

## Critères de validation
- [ ] Après une semaine de prod, `git status` reste propre sur le serveur.

_Référence : `docs/audit/AUDIT.md`_
