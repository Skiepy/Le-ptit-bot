# Revue de code et de sécurité — Le p'tit bot

Revue faite le 2026-10-03 sur `NozyZy/Le-ptit-bot@205424b` (fork `Skiepy/Le-ptit-bot` synchronisé).
Périmètre : `bot.py`, `fonctions.py`, `cogs/pokemon_starter.py`, fichiers de données, historique git.

Contexte important : le lien d'invitation du bot (README, `/invite`) demande `permissions=8`,
soit **Administrateur**. Toute faille qui fait agir le bot hérite donc des droits admin sur le serveur.

## Critique / élevé

### 1. `--clear` : aucune vérification de permission (`bot.py:1765`)
N'importe quel membre peut supprimer jusqu'à 1000 messages dans n'importe quel salon où il peut écrire.
Le bot étant admin, il supprime aussi les messages des autres (modération, annonces…).
**Correctif :** `@commands.has_permissions(manage_messages=True)` + `@commands.guild_only()`,
et utiliser `ctx.channel.purge(limit=nombre + 1)` au lieu de supprimer message par message.

### 2. Injection de `@everyone` / `@here` (aucun `allowed_mentions` dans tout le projet)
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
- `--calcul 9^9999999` : ~7 s de calcul bloquant mesurés (puis `str()` plante au-delà de 4300 chiffres) ;
  `9^999999999` = plusieurs minutes et des centaines de Mo de RAM. Le cooldown (3/5 s) ne protège pas.
- `--isPrime` / `/isprime` : la borne est `1e29`, mais `is_prime` est en O(√n) **avec 3 `print` par itération**
  (`fonctions.py:30`). Un premier ~1e28 = ~1e13 itérations → bot gelé pendant des jours.
  `/isprime` n'a en plus aucun cooldown. Mesuré : un premier de 1e14 = 5 millions de lignes imprimées.
- `--randomWord 100000000` : aucune borne, aucune cooldown → boucle géante + message énorme.
- `--dhcp 0.0.0.0/0` : `[str(ip) for ip in network]` matérialise 4 milliards d'adresses → OOM.
  `10.0.0.0/8` = 16 M de chaînes.
- `--prime` : relit `txt/primes.txt` (97 Mo) en entier en mémoire à chaque appel (`readlines()`).
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

- `/unban` (`bot.py:2484`) : supprime un élément de la liste pendant l'itération → **l'ID de salon
  suivant est perdu** (testé : retirer `2` de `[1,2,3]` laisse `[1]`).
- `--repeat` (`bot.py:1944`) : `for message in ctx.channel.history(...)` sur un itérateur async
  → `TypeError`, la commande ne marche pas.
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

- `txt/primes.txt` fait **97 Mo** (limite GitHub : 100 Mo, avertissement dès 50 Mo) ; 7 versions de
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
