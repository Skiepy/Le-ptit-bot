# [20] Mèmes `--master` / `--presentation` : fichier partagé entre utilisateurs

**Priorité :** P2  
**Vérification :** 📖 Lecture du code  
**Où :** `bot.py:2388`, `bot.py:2431`

## Problème
L'image générée est écrite dans `images/mastermeme.jpg` / `images/presentationmeme.png` (fichiers suivis par git), puis relue pour l'envoi. Si deux personnes lancent la commande en même temps, l'une peut recevoir l'image de l'autre. Ça modifie aussi des fichiers versionnés en prod.

## Preuve
Déduit de la lecture du code.

## Correctif proposé
Générer dans un `io.BytesIO` et envoyer `discord.File(buf, filename=...)`, comme `/sexestats` le fait déjà (l.1935). Retirer les deux fichiers du dépôt.

## Critères de validation
- [ ] Aucun fichier n'est écrit sur disque par ces commandes.

_Référence : `docs/audit/AUDIT.md`_
