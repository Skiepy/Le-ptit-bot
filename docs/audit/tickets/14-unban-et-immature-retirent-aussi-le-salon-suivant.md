# [14] `/unban` et `/immature` retirent aussi le salon suivant de la liste

**Priorité :** P2  
**Vérification :** ✅ Exécuté  
**Où :** `bot.py:2484-2495` (`/unban`), `bot.py:2547-2558` (`/immature`)

## Problème
La boucle fait `bansLines.remove(id)` pendant qu'elle parcourt `bansLines`. L'élément suivant est sauté, donc jamais réécrit dans le fichier.

## Preuve
Appel réel de `/unban` : `bans.txt` = `111 222 333 444`, unban du salon 222 → le fichier contient `111 444`. Le salon 333 est débanni sans que personne ne l'ait demandé. `/immature` utilise exactement le même code (non exécuté).

## Correctif proposé
Écrire la liste filtrée : `f.writelines(l for l in lines if l != chanID)`.

## Critères de validation
- [ ] Unban de 222 parmi `111 222 333 444` → `111 333 444`.

_Référence : `docs/audit/AUDIT.md`_
