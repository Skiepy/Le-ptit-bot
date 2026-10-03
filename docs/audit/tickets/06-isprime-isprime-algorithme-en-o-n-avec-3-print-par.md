# [06] `--isPrime` / `/isprime` : algorithme en O(√n) avec 3 `print` par tour

**Priorité :** P0  
**Vérification :** ✅ Exécuté  
**Où :** `fonctions.py:23-39`, `bot.py:2251`, `bot.py:2265`

## Problème
`is_prime` teste les diviseurs jusqu'à √n et fait 3 `print` à chaque itération, de façon synchrone. La borne acceptée est 1e29. `/isprime` n'a aucun cooldown.

## Preuve
Mesuré : un premier proche de 1e14 → 3,2 s de gel et 5 millions de lignes imprimées. Pour un premier proche de 1e28, c'est 10⁷ fois plus d'itérations, soit de l'ordre d'un an (extrapolé). Les `print` remplissent aussi les logs (journald / fichier) du LXC.

## Correctif proposé
- Supprimer les `print`.
- Utiliser `sympy.isprime`, ou un Miller-Rabin déterministe (exact jusqu'à 3,3·10²⁴ avec les 13 premiers nombres premiers comme bases ; abaisser la borne à 1e24 dans ce cas).
- Ajouter un cooldown à `/isprime`.

## Critères de validation
- [ ] `--isPrime 99999999999999999989` répond en moins de 10 ms.
- [ ] Plus aucune sortie `print` dans les logs.

_Référence : `docs/audit/AUDIT.md`_
