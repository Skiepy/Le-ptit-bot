# [05] `--calcul` : un exposant énorme gèle le bot pendant des minutes

**Priorité :** P0  
**Vérification :** ✅ Exécuté  
**Où :** `bot.py:2042-2117`

## Problème
`nb1 ** nb2` est calculé sans borne, de façon synchrone, dans la boucle asyncio. Pendant le calcul, le bot ne répond plus nulle part. Le cooldown (3 appels / 5 s par utilisateur) ne protège pas, car un seul appel suffit.

## Preuve
Mesuré sur la vraie commande :
- `--calcul 9^9999999` : 6,6 s de gel.
- `--calcul 9^99999999` : **223 s de gel**, +166 Mo.

Le résultat n'est même pas affichable : `str()` lève `ValueError` au-delà de 4300 chiffres.

## Correctif proposé
- Refuser si `nb2 * log10(nb1) > 4000` (le résultat dépasserait la limite d'affichage Discord).
- Borner aussi `*` par la taille du résultat.
- Remplacer `facto()` récursif par `math.factorial` (garder la borne 806 : 806! fait 1995 chiffres, juste sous la limite Discord).

## Critères de validation
- [ ] `--calcul 9^99999999` répond immédiatement par un refus.
- [ ] `--calcul 2^10` répond `1024`.

_Référence : `docs/audit/AUDIT.md`_
