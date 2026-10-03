# [10] Appels HTTP bloquants dans la boucle asyncio

**Priorité :** P1  
**Vérification :** 📖 Lecture du code  
**Où :** `bot.py` (kanye, skin, `/chat` l.3541, `/activity`), `cogs/pokemon_starter.py:276`, `:288`, `:851`, `:889`

## Problème
Tous les appels réseau utilisent `requests.get`, qui est synchrone : chaque appel gèle tout le bot le temps de la réponse. `/chat` n'a même pas de `timeout`, donc un serveur lent peut bloquer le bot indéfiniment.

## Preuve
Déduit de la lecture du code.

## Correctif proposé
- Passer à `aiohttp` (déjà installé avec discord.py), ou envelopper avec `await asyncio.to_thread(requests.get, ...)`.
- Mettre un `timeout` partout.

## Critères de validation
- [ ] Aucun `requests.get` direct dans une coroutine.
- [ ] `/chat` a un timeout.

_Référence : `docs/audit/AUDIT.md`_
