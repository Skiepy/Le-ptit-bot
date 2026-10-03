# [07] Flags du CTF en clair dans le dépôt public

**Priorité :** P0  
**Vérification :** ✅ Exécuté  
**Où :** `bot.py:2686-2705`

## Problème
`FLAG` et `FLAG2` (`CYBN{…}`) sont écrits en dur dans le code d'un dépôt public : il suffit de lire GitHub pour valider le challenge.

De plus, `/flag` appelle `send_message` deux fois dès qu'un joueur y a droit (la branche `else` s'exécute toujours dans ce cas).

## Preuve
Appel réel de `/flag` : le flag est bien envoyé, puis la 2ᵉ réponse lève `InteractionResponded`. Testé pour un joueur à 3 victoires et pour un joueur avec un match nul.

## Correctif proposé
- Lire les flags depuis `.env` (`os.getenv("CTF_FLAG")`).
- **Changer les flags** côté CTF : les actuels sont publics depuis longtemps.
- Restructurer `/flag` en `if/elif/else` avec une seule réponse (les deux flags dans un seul message si besoin).

## Critères de validation
- [ ] `grep CYBN bot.py` ne renvoie rien.
- [ ] `/flag` ne lève plus d'exception.

_Référence : `docs/audit/AUDIT.md`_
