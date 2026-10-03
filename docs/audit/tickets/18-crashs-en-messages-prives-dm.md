# [18] Crashs en messages privés (DM)

**Priorité :** P2  
**Vérification :** 📖 Lecture du code  
**Où :** `bot.py:399` et toutes les lignes `message.guild.name` de `on_message`, `--appel`, `--rename`, `--clear`, `/combat` (cog l.1139)

## Problème
En DM, `message.guild` vaut `None`. `on_message` utilise `message.guild.name` dans les logs dès qu'un mot nouveau est vu, donc `AttributeError`. Les commandes qui utilisent `guild_permissions` ou `interaction.guild.id` plantent aussi.

## Preuve
Déduit de la lecture du code.

## Correctif proposé
- Ignorer les DM en tête de `on_message` (ou utiliser `guild_name = message.guild.name if message.guild else "DM"`).
- Ajouter `@commands.guild_only()` / `@app_commands.guild_only()` sur les commandes de serveur.

## Critères de validation
- [ ] Envoyer un DM au bot ne produit aucune trace d'exception.

_Référence : `docs/audit/AUDIT.md`_
