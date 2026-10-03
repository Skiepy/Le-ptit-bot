# [12] Contenu des conversations et données des membres publiés sur GitHub

**Priorité :** P1  
**Vérification :** 📖 Lecture du code  
**Où :** `bot.py:376-406`, `txt/dico.txt`, `txt/leaderboard.txt`, `txt/pve.txt`, `txt/server_names.txt`, `data/`

## Problème
Le bot lit tous les messages de tous les serveurs et ajoute chaque mot nouveau à `txt/dico.txt`. Ce fichier est ensuite commité sur le dépôt public (commits « prod »). Les logs associent pseudo + serveur + mot. Les leaderboards (ID Discord + pseudo) sont aussi publics.

## Preuve
Constaté dans l'historique : `dico.txt` passe de 33 912 lignes (commit `29e4dd1`, mai 2026) à 39 779 lignes aujourd'hui, via les commits « prod » de l'auteur « maybe push in prod ».

## Correctif proposé
- Ne plus versionner les fichiers alimentés en production (voir ticket hygiène).
- Prévenir les serveurs que leurs messages alimentent un dictionnaire, ou désactiver la collecte.
- Arrêter de logger le mot avec le pseudo.

## Critères de validation
- [ ] Plus aucun fichier modifié par le bot en prod n'est suivi par git.

_Référence : `docs/audit/AUDIT.md`_
