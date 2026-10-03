# Revue de code et de sécurité — Le p'tit bot

Revue faite le 2026-10-03 sur `NozyZy/Le-ptit-bot@205424b` (fork `Skiepy/Le-ptit-bot` synchronisé).
Périmètre : `bot.py`, `fonctions.py`, `cogs/pokemon_starter.py`, fichiers de données, historique git.

**Méthode de vérification.** Les points marqués ✅ *exécuté* ont été reproduits en appelant les vraies fonctions
de `bot.py` (discord.py 2.7.1, Python 3.11) avec des objets Discord simulés (`unittest.mock`), sur une copie
du dépôt. Le bot n'a **pas** été connecté à un vrai serveur Discord. Les points marqués 📖 *lecture* sont
déduits de la lecture du code, sans exécution. Les durées extrapolées sont indiquées comme telles.

Contexte important : le lien d'invitation du bot (README, `/invite`) demande `permissions=8`,
soit **Administrateur**. Toute faille qui fait agir le bot hérite donc des droits admin sur le serveur.

## Critique / élevé

### 1. `--clear` : aucune vérification de permission (`bot.py:1765`) — ✅ exécuté
`clear.checks == []`, aucun `bot.check` global, et `on_message` transmet bien les commandes à `process_commands`.
Appel réel avec un membre sans `manage_messages` ni `administrator` : `--clear 50` → 51 messages supprimés.
N'importe quel membre peut supprimer jusqu'à 1000 messages dans n'importe quel salon où il peut écrire.
Le bot étant admin, il supprime aussi les messages des autres (modération, annonces…).
**Correctif :** `@commands.has_permissions(manage_messages=True)` + `@commands.guild_only()`,
et utiliser `ctx.channel.purge(limit=nombre + 1)` au lieu de supprimer message par message.

### 2. Injection de `@everyone` / `@here` (aucun `allowed_mentions` dans tout le projet) — 📖 lecture
Non testé sur un vrai serveur : le ping effectif dépend de la permission « Mentionner @everyone » du bot
(incluse dans Administrateur).
Le bot renvoie du texte contrôlé par les utilisateurs sans filtrer les mentions. Avec la permission
Administrateur, il peut pinguer tout le serveur :
- `--addInsult @everyone …` (ouvert à tous) → stocké dans `txt/insultes.txt`, puis renvoyé à chaque « tg ».
  C'est **persistant** : n'importe qui peut piéger le bot pour des pings ultérieurs.
- `/addquidenous` (ouvert à tous) → stocké dans `txt/nous.txt`, renvoyé pendant les parties.
- `/ask`, `--crypt` (texte original renvoyé), `--poll`, `--repeat`, `--rename` (texte du nom renvoyé).

**Correctif :** une ligne globale suffit :
`commands.Bot(..., allowed_mentions=discord.AllowedMentions(everyone=False, roles=False, users=True))`
(et retirer `@everyone` codé en dur ligne ~876 si on veut être cohérent).

### 3. Déni de service : opérations synchrones qui gèlent tout le bot
discord.py est mono-thread (asyncio) : un calcul long bloque **tous** les serveurs.
Mesures ✅ exécutées (vraie commande + tâche témoin toutes les 0,1 s pour mesurer le gel de la boucle) :

| Commande | Gel mesuré | Commentaire |
|---|---|---|
| `--calcul 9^9999999` | 6,6 s | puis `ValueError` dans `str()` (> 4300 chiffres). Un exposant plus grand = plus long (extrapolé, non testé). |
| `--isPrime 100000000000031` (1e14) | 3,2 s | O(√n) + 3 `print` par itération. La borne autorisée est 1e29 : un premier ~1e28 ≈ 10⁷ × plus long, soit de l'ordre d'un an (extrapolé). `/isprime` n'a aucun cooldown. |
| `--dhcp 10.0.0.0/10` | 4,7 s | 4 M d'adresses. `/8` ≈ 4× plus, `/0` ≈ 1000× plus + saturation mémoire (extrapolé). |
| `--randomWord 3000000` | 1,2 s | Plutôt mineur : linéaire, et le message de 25 M caractères est de toute façon refusé par Discord. Pas de borne ni de cooldown. |

- `--prime` (📖) : relit `txt/primes.txt` (97 Mo) en entier en mémoire à chaque appel, **uniquement pour lire
  sa dernière ligne** (voir « Hygiène du dépôt »).
- Tous les `requests.get(...)` (kanye, skin, chat, activity, pokémon…) sont bloquants dans la boucle
  asyncio ; `/chat` n'a même pas de `timeout`.

**Correctif :** borner les entrées (exposant, nb de mots, taille du réseau, nb ≤ 1e12 ou test de Miller-Rabin),
supprimer les `print` de `is_prime`, passer en `aiohttp` (déjà dépendance de discord.py) ou `asyncio.to_thread`.

### 4. Flags CTF en clair dans le code public (`bot.py:2686`)
`FLAG` et `FLAG2` (`CYBN{…}`) sont dans le dépôt public : le challenge est résolu par un simple `git clone`.
**Correctif :** les mettre dans `.env`. De plus `/flag` appelle `send_message` deux fois → `InteractionResponded`.

### 5. Tokens Discord dans l'historique git
13 tokens distincts du bot (`NjUzNTYz…`) sont présents dans l'historique (créés le 2019-12-09,
derniers commits concernés en 2021-07). Ils sont probablement révoqués depuis (le token actuel est
lu depuis `.env`), mais **à confirmer** dans le portail développeur Discord : régénérer le token s'il y a
le moindre doute. `txt/admin.txt` et `txt/names.txt` ont aussi été commités en 2020 puis retirés,
ils restent lisibles dans l'historique.

## Moyen

### 6. `Intents.all()` + collecte de tous les messages (`bot.py:59`, `bot.py:383`)
Le bot lit chaque message de chaque serveur et enregistre les mots nouveaux dans `txt/dico.txt`,
fichier ensuite **commité publiquement** (« prod » commits). Combiné aux logs `user - serveur - mot`,
c'est une fuite de contenu de conversations privées vers un dépôt public. Idem pour `leaderboard.txt`,
`pve.txt`, `server_names.txt` (IDs + pseudos) commités.

### 7. Écritures de fichiers sans verrou ni limite
`txt/insultes.txt`, `nous.txt`, `dico.txt`, `leaderboard.txt` sont lus/réécrits à chaque message,
sans verrou ni limite de taille : spam = croissance illimitée du disque, et risque de corruption
(deux messages simultanés réécrivant `dico.txt`). `dico.txt` est relu intégralement (340 Ko)
**à chaque message reçu** et la recherche `not in dicoLines` est en O(n) par mot.

### 8. Leaderboard fragile (`bot.py:3279` et suivantes)
- Format `id-wins-loses-ratio-name` : un pseudo contenant `-` décale les colonnes.
- Recherche par sous-chaîne `str(id) in ligne` : le pseudo d'un joueur peut contenir l'ID d'un autre
  et lui « voler » sa ligne (utile notamment pour `/flag` qui lit `pve.txt`).
**Correctif :** passer en JSON indexé par ID.

## Bugs fonctionnels relevés au passage

- `/unban` (`bot.py:2484`) — ✅ exécuté : supprime un élément de la liste pendant l'itération → **l'ID de salon
  suivant est perdu**. Appel réel : `bans.txt` = `111 222 333 444`, `/unban` dans le salon 222 → `111 444`
  (le salon 333 est débanni sans que personne ne l'ait demandé).
- `--repeat` (`bot.py:1944`) — ✅ exécuté avec discord.py 2.7.1 : `history()` renvoie un générateur async,
  `for` dessus lève `TypeError: 'async_generator' object is not iterable` avant le `send`.
  Si la commande fonctionne en prod, le bot en prod ne fait pas tourner ce code-là (version différente).
- Compatibilité : la dépendance `Tyradex` (via `pratik`) utilise une syntaxe f-string réservée à
  **Python ≥ 3.12** ; l'import échoue en 3.11. Le README indique Python 3.10.2.
- `--prime` : le `return` sur « Primo no » ne décrémente pas `nbprime` → après 3 appels,
  la commande répond « je suis occupé » pour toujours (jusqu'au redémarrage).
- `logger.info(nb, biggest, n_max)` et `logger.info("…:", mot)` : mauvais usage du logger → erreurs de format.
- Messages privés : `message.guild.name` dans `on_message` → `AttributeError` dès qu'un DM contient un mot nouveau ;
  `--appel`/`--rename` en DM → `guild_permissions` inexistant.
- `discord.Client(intents=intents)` créé ligne 61 mais jamais utilisé.
- `facto()` récursif limité à 806 : `math.factorial` suffit.
- `--dhcp` : plus de participants que d'IP → `IndexError` sur `ips.pop(0)`.
- `requirements.txt` : `numpy` non épinglé.

## Hygiène du dépôt

- `txt/primes.txt` (10,4 M de nombres premiers, jusqu'à 187 465 331) fait **97 Mo**. Il n'est jamais envoyé :
  `--prime` envoie `txt/prime.txt` (7 Mo, premiers jusqu'à 14 064 991) et ne lit `primes.txt` que pour en
  extraire le dernier nombre. Le code qui l'alimentait est commenté. Il fait (limite GitHub : 100 Mo, avertissement dès 50 Mo) ; 7 versions de
  83 à 97 Mo dans l'historique → pack de 67 Mo. Ce fichier se régénère en quelques secondes : il ne devrait pas être versionné.
- 3174 fichiers d'un `venv/` ont été commités dans le passé.
- `.gitignore` de 961 lignes (template générique), alors que les fichiers d'état du bot (`txt/tg.txt`,
  `leaderboard.txt`, `onecops_counter.txt`, `dico.txt`…) sont, eux, versionnés et modifiés en prod.
- Bits exécutables sur des images, le README, la licence… (corrigé sur cette branche).

## Priorités recommandées

1. Ajouter `allowed_mentions` global + permission sur `--clear` (5 minutes, ferme les deux pires failles).
2. Borner `calcul`, `isPrime`, `randomWord`, `dhcp`; retirer les `print` de `is_prime`.
3. Sortir les flags du code, vérifier/régénérer le token.
4. Réduire la permission demandée à l'invitation (pas besoin d'Administrateur).
5. Sortir les fichiers d'état et `primes.txt` du versionnement (à faire **sur le dépôt original**,
   sinon le fork divergera à chaque synchro).
