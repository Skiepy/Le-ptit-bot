# [04] `--prime` charge 97 Mo en mémoire à chaque appel et peut se bloquer définitivement

**Priorité :** P0  
**Vérification :** ✅ Exécuté  
**Où :** `bot.py:2199-2246`, `txt/primes.txt`

## Problème
`--prime` lit tout `txt/primes.txt` (97 Mo, 10,4 M de lignes) avec `readlines()`, **uniquement pour récupérer le dernier nombre**. Le fichier envoyé à l'utilisateur est un autre fichier : `txt/prime.txt` (7 Mo).

En plus, le `return` sur la branche « Primo no » ne décrémente pas `nbprime`. Après 3 passages par cette branche, la commande répond « je suis occupé » jusqu'au redémarrage.

`logger.info(nb, biggest, n_max)` est aussi un mauvais appel du logger : il affiche une erreur de formatage sur stderr.

## Preuve
Mesuré : **+720 Mo de RAM** et 3,2 s de gel par appel. Jusqu'à 3 appels simultanés sont autorisés, soit environ 2,2 Go (extrapolé). Le compteur bloqué vient de la lecture du code.

## Correctif proposé
- Remplacer la lecture par une constante `BIGGEST_PRIME = 187465331`, et retirer `txt/primes.txt` du dépôt (à faire upstream, voir ticket hygiène).
- Utiliser `try/finally` autour du compteur, ou un `asyncio.Semaphore(1)`.
- Corriger le `logger.info`.

## Critères de validation
- [ ] `--prime 5` n'augmente plus la RAM de façon notable.
- [ ] Après 10 appels sur la branche « Primo no », la commande répond toujours normalement.

_Référence : `docs/audit/AUDIT.md`_
