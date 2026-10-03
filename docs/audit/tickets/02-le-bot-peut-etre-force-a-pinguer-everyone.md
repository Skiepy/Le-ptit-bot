# [02] Le bot peut être forcé à pinguer `@everyone`

**Priorité :** P0  
**Vérification :** 📖 Lecture du code  
**Où :** `commands.Bot(...)` (`bot.py:62`) ; vecteurs : `bot.py:420`, `bot.py:431`, `bot.py:1141`, `bot.py:1963`, `bot.py:2122`, `bot.py:3459`

## Problème
Aucun `allowed_mentions` dans le projet. Le bot a la permission Administrateur, donc tout texte utilisateur qu'il renvoie peut contenir un `@everyone` / `@here` / `<@&rôle>` qui pingue réellement.

Vecteurs persistants (le texte est stocké puis renvoyé plus tard à d'autres personnes) :
- `--addInsult @everyone …` → `txt/insultes.txt`, renvoyé quand quelqu'un écrit « tg ».
- `--addBranlette jme @everyone …` → `txt/branlette.txt`, renvoyé sur « branle ».

Vecteurs directs : `/ask`, `--poll`, `--crypt` (le texte original est renvoyé).

Pas un vecteur : `/addquidenous`, car les questions sont affichées dans un embed, et Discord ne pingue jamais depuis un embed.

## Preuve
Non testé sur un vrai serveur. Le ping effectif dépend de la permission « Mentionner @everyone », incluse dans Administrateur.

## Correctif proposé
- Passer `allowed_mentions=discord.AllowedMentions(everyone=False, roles=False, users=True, replied_user=True)` à `commands.Bot(...)`.
- Pour les rares messages qui doivent vraiment pinguer (`bot.py:876`, niveau 100 dans le cog), passer explicitement `allowed_mentions=discord.AllowedMentions(everyone=True)` sur ce `send` uniquement.
- Nettoyer les entrées déjà présentes dans `insultes.txt` / `branlette.txt` qui contiendraient `@everyone`/`@here`.

## Critères de validation
- [ ] `--addInsult @everyone test` puis « tg » n'envoie aucune notification.
- [ ] Les pings voulus (feur, niveau 100) fonctionnent toujours.

_Référence : `docs/audit/AUDIT.md`_
