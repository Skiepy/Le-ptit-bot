# [22] Dépendances : Pillow vulnérable, Python ≥ 3.12 requis, numpy non épinglé

**Priorité :** P2  
**Vérification :** ✅ Exécuté  
**Où :** `requirements.txt`, `README.md`

## Problème
- `pip-audit` signale **13 vulnérabilités** dans `pillow 12.2.0`, toutes corrigées en 12.3.0. L'exposition est faible : le bot n'ouvre que ses propres images et des images de GitHub/Wikimedia, pas des fichiers fournis par les utilisateurs.
- `Tyradex` (via `pratik`) utilise une syntaxe f-string réservée à **Python ≥ 3.12** : l'import échoue en 3.11. Le README annonce Python 3.10.2.
- `numpy` n'a pas de version épinglée.

## Preuve
`pip-audit -r requirements.txt` exécuté. L'échec d'import a été constaté en Python 3.11.15.

## Correctif proposé
- `pillow~=12.3.0`, épingler `numpy`.
- Mettre à jour le README (Python 3.12+).

## Critères de validation
- [ ] `pip-audit -r requirements.txt` ne remonte rien.
- [ ] Le README indique la bonne version de Python.

_Référence : `docs/audit/AUDIT.md`_
