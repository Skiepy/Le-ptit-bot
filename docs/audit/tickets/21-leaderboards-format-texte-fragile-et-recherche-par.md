# [21] Leaderboards : format texte fragile et recherche par sous-chaîne

**Priorité :** P2  
**Vérification :** 📖 Lecture du code  
**Où :** `bot.py:3269-3370`, `/flag`

## Problème
Lignes `id-victoires-défaites-ratio-nom`. `getScoreLeaderBoard` et `getPlaceLeaderbord` cherchent avec `str(id) in ligne`, une recherche par sous-chaîne. Un nom d'utilisateur (chiffres autorisés) égal à l'ID d'un autre joueur peut faire renvoyer la mauvaise ligne, ce qui compte pour `/flag`. Pas d'écriture atomique non plus (cf. ticket écritures).

## Preuve
Déduit de la lecture du code. Les noms d'utilisateur Discord actuels n'acceptent pas `-`, donc le décalage de colonnes est peu probable (correction de ma première analyse).

## Correctif proposé
Passer à un JSON `{user_id: {wins, losses, name}}`, avec écriture atomique.

## Critères de validation
- [ ] Le score est retrouvé uniquement par égalité exacte de l'ID.

_Référence : `docs/audit/AUDIT.md`_
