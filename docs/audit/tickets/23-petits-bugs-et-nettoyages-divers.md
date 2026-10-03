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
- **Coût de chaque message** : `on_message` relit 4 fichiers dont `dico.txt` (40 000 mots), puis trie l'ensemble, **à chaque message reçu**. Mesuré : 18 ms de travail bloquant par message banal, soit environ 55 messages/s au maximum pour tout le bot. Garder le dico en mémoire dans un `set` et n'écrire que les ajouts.
- `--join` / `--leave` : `AttributeError` si l'auteur n'est pas en vocal ; n'importe qui peut faire entrer le bot dans un salon vocal.
- `/delete_starter` : un double clic sur « Supprimer » fait `pop()` deux fois, d'où un `KeyError` au second clic.

## Preuve
`randomWord` et le coût de `on_message` exécutés. Le reste vient de la lecture du code.

## Correctif proposé
Corriger au cas par cas.

## Critères de validation
- [ ] Chaque point de la liste est corrigé ou explicitement abandonné.

_Référence : `docs/audit/AUDIT.md`_
