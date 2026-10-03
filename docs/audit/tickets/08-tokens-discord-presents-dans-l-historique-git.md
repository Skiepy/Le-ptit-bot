# [08] Tokens Discord présents dans l'historique git

**Priorité :** P0  
**Vérification :** ✅ Exécuté  
**Où :** Historique git (commits de 2019 à 2021-07)

## Problème
13 tokens distincts du bot (préfixe `NjUzNTYz…`, émis le 2019-12-09) sont lisibles dans l'historique public. `txt/admin.txt` et `txt/names.txt` ont aussi été commités en 2020, puis retirés.

## Preuve
Trouvés par recherche dans `git log --all -p`. **Révocation non vérifiable de mon côté** : il faut le portail développeur Discord.

## Correctif proposé
- Vérifier dans le portail développeur que le token actuel n'est pas l'un d'eux. En cas de doute, cliquer sur « Reset Token ».
- Ne pas réécrire l'historique pour ça : un token révoqué est inoffensif, et la réécriture casserait tous les forks.

## Critères de validation
- [ ] Le propriétaire du bot confirme que le token actuel a été généré après 2021-07.

_Référence : `docs/audit/AUDIT.md`_
