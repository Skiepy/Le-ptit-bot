# [15] `--repeat` plante systématiquement

**Priorité :** P2  
**Vérification :** ✅ Exécuté  
**Où :** `bot.py:1944-1950`

## Problème
`ctx.channel.history()` renvoie un générateur async. Le parcourir avec un `for` classique lève `TypeError`, avant même l'envoi du message.

## Preuve
Appel réel avec discord.py 2.7.1 : `TypeError: 'async_generator' object is not iterable`, `send` jamais appelé. (Yanis indique que la commande marche chez lui : à vérifier si la prod tourne bien sur ce code.)

## Correctif proposé
Remplacer par `await ctx.message.delete()`, puis `await ctx.send(...)`.

## Critères de validation
- [ ] `--repeat salut` supprime le message d'origine et renvoie « salut ».

_Référence : `docs/audit/AUDIT.md`_
