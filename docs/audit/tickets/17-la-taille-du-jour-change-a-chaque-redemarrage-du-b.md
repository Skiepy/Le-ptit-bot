# [17] La taille « du jour » change à chaque redémarrage du bot

**Priorité :** P2  
**Vérification :** ✅ Exécuté  
**Où :** `bot.py:694`

## Problème
`hash((user.id, "YYYY-MM-DD"))` contient une chaîne. Or Python randomise le hash des chaînes à chaque lancement (`PYTHONHASHSEED`). La valeur « du jour » n'est donc stable que jusqu'au prochain redémarrage. Les stats de `/sexestats` restent cohérentes, mais l'affichage ne l'est pas.

## Preuve
Même utilisateur, même date, 3 lancements de Python : tailles 12, 26 et 4.

## Correctif proposé
Utiliser une graine stable : `int(hashlib.sha256(f"{user.id}:{date}".encode()).hexdigest(), 16)` et `random.Random(seed)` (au lieu de reseeder le `random` global).

## Critères de validation
- [ ] Deux lancements du bot donnent la même taille pour le même utilisateur le même jour.

_Référence : `docs/audit/AUDIT.md`_
