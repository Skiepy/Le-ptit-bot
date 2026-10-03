# [16] Erreur Pokémon du jour : le message d'erreur n'est jamais envoyé

**Priorité :** P2  
**Vérification :** 📖 Lecture du code  
**Où :** `bot.py:835-845`

## Problème
Dans le `except`, la première instruction est `await channel.send()` sans contenu. L'API Discord refuse les messages vides (erreur 50006). L'exception est donc relancée depuis le `except`, et les deux messages d'erreur suivants ne partent jamais.

## Preuve
Déduit de la lecture du code et du comportement de l'API (non testé contre Discord).

## Correctif proposé
Supprimer `await channel.send()` et le doublon du message.
Optionnel : ne pas afficher `str(e)` aux utilisateurs.

## Critères de validation
- [ ] Quand le cache Pokémon est vide, l'image d'erreur et le message s'affichent.

_Référence : `docs/audit/AUDIT.md`_
