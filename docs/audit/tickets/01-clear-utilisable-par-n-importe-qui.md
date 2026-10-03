# [01] `--clear` utilisable par n'importe qui

**Priorité :** P0  
**Vérification :** ✅ Exécuté  
**Où :** `bot.py:1765-1779`

## Problème
La commande `--clear N` n'a aucune vérification de permission. Il n'y a pas non plus de `bot.check` global. N'importe quel membre peut faire supprimer par le bot jusqu'à 1000 messages, ceux des autres compris, dans tout salon où il peut écrire.

## Preuve
Appel de la vraie commande avec un auteur sans `manage_messages` ni `administrator` : `clear.checks == []`, et `--clear 50` supprime 51 messages.

## Correctif proposé
- Ajouter `@commands.has_permissions(manage_messages=True)` et `@commands.guild_only()`.
- Remplacer la boucle de `delete()` par `await ctx.channel.purge(limit=nombre + 1)`.
- Gérer `commands.MissingPermissions` dans `on_command_error` pour répondre proprement.

## Critères de validation
- [ ] Un membre sans « Gérer les messages » reçoit un refus et rien n'est supprimé.
- [ ] Un modérateur peut toujours utiliser la commande.

_Référence : `docs/audit/AUDIT.md`_
