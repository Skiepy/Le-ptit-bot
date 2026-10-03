# [11] Permissions excessives : Administrateur demandé à l'invitation + `Intents.all()`

**Priorité :** P1  
**Vérification :** 📖 Lecture du code  
**Où :** `README.md`, `bot.py:49`, `bot.py:59-61`, commande `/invite` (`bot.py:2561`)

## Problème
Le lien d'invitation demande `permissions=8` (Administrateur). Toute faille du bot (cf. `--clear`, `@everyone`) s'exécute donc avec les pleins pouvoirs. `Intents.all()` active aussi les intents privilégiés, dont la présence, qui n'est pas utilisée.

`client = discord.Client(intents=intents)` (l.61) est créé mais jamais utilisé.

## Preuve
Déduit de la lecture du code.

## Correctif proposé
- Calculer les permissions réellement nécessaires (envoyer des messages, embeds, fichiers, réactions, gérer les messages, threads, pseudo, vocal) et régénérer le lien.
- Remplacer `Intents.all()` par `Intents.default()` + `message_content` + `members`.
- Supprimer `client`.
- Sur les serveurs existants, retirer le rôle admin du bot.

## Critères de validation
- [ ] Le lien d'invitation ne contient plus `permissions=8`.
- [ ] Le bot fonctionne sans le rôle Administrateur.

_Référence : `docs/audit/AUDIT.md`_
