# [23] Petits bugs et nettoyages divers

**Priorité :** P2  
**Vérification :** 📖 Lecture du code  
**Où :** Voir la liste

## Problème
- `--randomWord N` : pas de borne ni de cooldown. Mesuré : N=3 000 000 → 1,2 s de gel, message de 25 M caractères refusé par Discord. Mineur.
- `logger.info("…:", mot)` (`bot.py:442`) et `logger.info(nb, biggest, n_max)` : mauvais usage du logger, ce qui affiche des erreurs de formatage.
- `facto()` récursif (`fonctions.py:100`) : remplacer par `math.factorial`.
- « Qui d'entre nous » : un menu de sélection Discord est limité à 25 options. Au-delà de 25 joueurs, la création du vote échoue.
- `/mature` et `/immature` sont définies avec les noms Python `ban` / `unban`, ce qui écrase les fonctions précédentes. Pas de bug fonctionnel (les commandes sont enregistrées à la décoration), mais c'est trompeur.
- `random.seed(seed)` puis `random.seed(None)` sur le générateur global (`bot.py:695`) : utiliser `random.Random(seed)`.

## Preuve
`randomWord` exécuté. Le reste vient de la lecture du code.

## Correctif proposé
Corriger au cas par cas.

## Critères de validation
- [ ] Chaque point de la liste est corrigé ou explicitement abandonné.

_Référence : `docs/audit/AUDIT.md`_
