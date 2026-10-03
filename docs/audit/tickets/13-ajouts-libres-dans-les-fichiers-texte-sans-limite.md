# [13] Ajouts libres dans les fichiers texte, sans limite ni modération

**Priorité :** P1  
**Vérification :** 📖 Lecture du code  
**Où :** `bot.py:420-443` (`--addInsult`, `--addBranlette`), `bot.py:4019` (`/addquidenous`)

## Problème
N'importe qui peut ajouter du texte arbitraire (jusqu'à 2000 caractères par message, 6000 pour une option de slash) dans des fichiers que le bot renvoie ensuite à tout le monde. Il n'y a ni limite de taille, ni cooldown, ni possibilité de retirer une entrée sans éditer le fichier à la main. Ces fichiers sont aussi partagés entre **tous les serveurs** : un serveur peut injecter du contenu qui s'affichera sur un autre.

Effets : spam, contenu choquant/haineux propagé, croissance du disque.

## Preuve
Déduit de la lecture du code.

## Correctif proposé
- Longueur max (ex. 200 caractères) et cooldown.
- Commande de suppression réservée aux admins.
- Optionnel : stockage par serveur.

## Critères de validation
- [ ] Une entrée de plus de 200 caractères est refusée.
- [ ] Un admin peut supprimer une entrée.

_Référence : `docs/audit/AUDIT.md`_
