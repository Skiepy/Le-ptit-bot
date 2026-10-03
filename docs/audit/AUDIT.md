# Audit de sécurité et revue de code — Le p'tit bot

| | |
|---|---|
| **Dépôt audité** | `NozyZy/Le-ptit-bot` @ `205424b` (fork `Skiepy/Le-ptit-bot` synchronisé) |
| **Date** | 2026-10-03 |
| **Périmètre** | `bot.py` (4070 l.), `fonctions.py`, `cogs/pokemon_starter.py` (1451 l.), `txt/`, `database/`, `requirements.txt`, historique git |
| **Résultat** | 24 points : 8 P0, 5 P1, 11 P2 |

## Sommaire
1. [Méthode et niveau de preuve](#1-méthode-et-niveau-de-preuve)
2. [Le risque principal : faire tomber le LXC](#2-le-risque-principal--faire-tomber-le-lxc)
3. [Tableau récapitulatif](#3-tableau-récapitulatif)
4. [Plan de correction conseillé](#4-plan-de-correction-conseillé)
5. [Détail des points](#5-détail-des-points)
6. [Corrections par rapport à la première revue](#6-corrections-par-rapport-à-la-première-revue)
7. [État du fork et du dépôt git](#7-état-du-fork-et-du-dépôt-git)
8. [Ce qui n'a pas été vérifié](#8-ce-qui-na-pas-été-vérifié)

## 1. Méthode et niveau de preuve

Chaque point indique comment il a été établi :

- **✅ Exécuté** : reproduit en appelant les **vraies fonctions** de `bot.py`, `fonctions.py` ou du cog, importés tels quels
  (discord.py 2.7.1, Python 3.11.15). Seuls les objets Discord (message, auteur, salon, interaction) sont simulés
  avec `unittest.mock`. Tout tourne sur une copie du dépôt. Les temps de gel sont mesurés par une tâche témoin qui tourne
  toutes les 0,1 s dans la même boucle asyncio. La RAM est le pic `ru_maxrss` du processus.
- **📖 Lecture du code** : déduit de la lecture, sans exécution.
- Les valeurs **extrapolées** sont signalées comme telles.

**Le bot n'a jamais été connecté à un vrai serveur Discord.** Tout ce qui dépend de la réponse de l'API Discord
(ping effectif d'un `@everyone`, refus d'un message vide) est déduit, pas observé.

Limite de l'environnement de test : la dépendance `Tyradex` ne s'importe pas en Python 3.11 (voir point 22).
Elle a été remplacée par un module vide pour les tests, ce qui n'affecte aucune des fonctions testées.

## 2. Le risque principal : faire tomber le LXC

Ton pote a raison : c'est le risque le plus concret. Il y a deux mécanismes différents.

**a) Saturation mémoire.** C'est ce qui peut tuer le processus, voire gêner le reste du conteneur. Mesures :

| Commande | Pic de RAM | Gel du bot |
|---|---|---|
| `--dhcp 10.0.0.0/8` | **+1,16 Go** | 19 s |
| `--dhcp 0.0.0.0/0` | ~300 Go (extrapolé) → OOM certain | — |
| `--prime 5` | **+720 Mo** par appel (3 appels simultanés autorisés) | 3,2 s |
| `--calcul 9^99999999` | +166 Mo | **223 s** |

Base du processus : environ 57 Mo. Sur un LXC à 1 ou 2 Go, une seule commande `--dhcp` suffit.

**b) Gel de la boucle asyncio.** Le bot est mono-thread : un calcul long fige le bot **sur tous les serveurs**, mais le LXC reste debout.
Exemples : `--calcul`, `--isPrime` (jusqu'à environ un an pour un grand nombre premier, extrapolé) et tous les `requests.get` synchrones.
Pendant un gel trop long, Discord coupe la connexion (le heartbeat n'est plus envoyé).

**c) Effet en cascade.** Si l'OOM killer tue le bot pendant qu'il écrit `data/pokemon_starters.json`, le fichier est tronqué.
Au redémarrage, le bot repart de `{}` et écrase le fichier : **toutes les données Pokémon sont perdues** (point 9).

**Protection côté infra, en attendant les correctifs.** Fixer une limite mémoire au service, par exemple dans une unité systemd :
`MemoryMax=512M` + `Restart=on-failure`. Comme ça, l'OOM tue le bot, pas le reste du conteneur. Ça ne remplace pas les correctifs.

## 3. Tableau récapitulatif

| # | Prio | Preuve | Point |
|---|---|---|---|
| [01](tickets/01-clear-utilisable-par-n-importe-qui.md) | P0 | ✅ | `--clear` utilisable par n'importe qui |
| [02](tickets/02-le-bot-peut-etre-force-a-pinguer-everyone.md) | P0 | 📖 | Le bot peut être forcé à pinguer `@everyone` |
| [03](tickets/03-dhcp-peut-saturer-la-ram-du-lxc.md) | P0 | ✅ | `--dhcp` peut saturer la RAM du LXC |
| [04](tickets/04-prime-charge-97-mo-en-memoire-a-chaque-appel-et-pe.md) | P0 | ✅ | `--prime` charge 97 Mo en mémoire à chaque appel et peut se bloquer définitivement |
| [05](tickets/05-calcul-un-exposant-enorme-gele-le-bot-pendant-des.md) | P0 | ✅ | `--calcul` : un exposant énorme gèle le bot pendant des minutes |
| [06](tickets/06-isprime-isprime-algorithme-en-o-n-avec-3-print-par.md) | P0 | ✅ | `--isPrime` / `/isprime` : algorithme en O(√n) avec 3 `print` par tour |
| [07](tickets/07-flags-du-ctf-en-clair-dans-le-depot-public.md) | P0 | ✅ | Flags du CTF en clair dans le dépôt public |
| [08](tickets/08-tokens-discord-presents-dans-l-historique-git.md) | P0 | ✅ | Tokens Discord présents dans l'historique git |
| [09](tickets/09-ecritures-de-fichiers-non-atomiques-un-crash-peut.md) | P1 | 📖 | Écritures de fichiers non atomiques : un crash peut effacer les données Pokémon |
| [10](tickets/10-appels-http-bloquants-dans-la-boucle-asyncio.md) | P1 | 📖 | Appels HTTP bloquants dans la boucle asyncio |
| [11](tickets/11-permissions-excessives-administrateur-demande-a-l.md) | P1 | 📖 | Permissions excessives : Administrateur demandé à l'invitation + `Intents.all()` |
| [12](tickets/12-contenu-des-conversations-et-donnees-des-membres-p.md) | P1 | 📖 | Contenu des conversations et données des membres publiés sur GitHub |
| [13](tickets/13-ajouts-libres-dans-les-fichiers-texte-sans-limite.md) | P1 | 📖 | Ajouts libres dans les fichiers texte, sans limite ni modération |
| [14](tickets/14-unban-et-immature-retirent-aussi-le-salon-suivant.md) | P2 | ✅ | `/unban` et `/immature` retirent aussi le salon suivant de la liste |
| [15](tickets/15-repeat-plante-systematiquement.md) | P2 | ✅ | `--repeat` plante systématiquement |
| [16](tickets/16-erreur-pokemon-du-jour-le-message-d-erreur-n-est-j.md) | P2 | 📖 | Erreur Pokémon du jour : le message d'erreur n'est jamais envoyé |
| [17](tickets/17-la-taille-du-jour-change-a-chaque-redemarrage-du-b.md) | P2 | ✅ | La taille « du jour » change à chaque redémarrage du bot |
| [18](tickets/18-crashs-en-messages-prives-dm.md) | P2 | 📖 | Crashs en messages privés (DM) |
| [19](tickets/19-pokemon-cooldowns-de-combat-et-d-xp-contournables.md) | P2 | 📖 | Pokémon : cooldowns de combat et d'XP contournables |
| [20](tickets/20-memes-master-presentation-fichier-partage-entre-ut.md) | P2 | 📖 | Mèmes `--master` / `--presentation` : fichier partagé entre utilisateurs |
| [21](tickets/21-leaderboards-format-texte-fragile-et-recherche-par.md) | P2 | 📖 | Leaderboards : format texte fragile et recherche par sous-chaîne |
| [22](tickets/22-dependances-pillow-vulnerable-python-3-12-requis-n.md) | P2 | ✅ | Dépendances : Pillow vulnérable, Python ≥ 3.12 requis, numpy non épinglé |
| [23](tickets/23-petits-bugs-et-nettoyages-divers.md) | P2 | 📖 | Petits bugs et nettoyages divers |
| [24](tickets/24-hygiene-du-depot-a-traiter-sur-le-depot-original.md) | P2 | ✅ | Hygiène du dépôt (à traiter sur le dépôt original) |

## 4. Plan de correction conseillé

1. **Tout de suite, sans toucher au code** : limite mémoire systemd sur le LXC ; vérifier ou réinitialiser le token (08) ; retirer le rôle Administrateur du bot sur les serveurs (11).
2. **Correctifs rapides, moins d'une heure, l'essentiel du risque** : `allowed_mentions` global (02), permission sur `--clear` (01),
   bornes sur `--dhcp` / `--calcul` / `--isPrime` / `--prime` (03-06), suppression des `print` d'`is_prime`.
3. **Fiabilité** : écritures atomiques (09), HTTP asynchrone (10), flags dans `.env` + changement des flags (07).
4. **Le reste** : bugs P2, puis hygiène du dépôt avec le propriétaire du dépôt original (24).

## 5. Détail des points

### 01. `--clear` utilisable par n'importe qui

**Priorité :** P0  
**Vérification :** ✅ Exécuté  
**Où :** `bot.py:1765-1779`

#### Problème
La commande `--clear N` n'a aucune vérification de permission. Il n'y a pas non plus de `bot.check` global. N'importe quel membre peut faire supprimer par le bot jusqu'à 1000 messages, ceux des autres compris, dans tout salon où il peut écrire.

#### Preuve
Appel de la vraie commande avec un auteur sans `manage_messages` ni `administrator` : `clear.checks == []`, et `--clear 50` supprime 51 messages.

#### Correctif proposé
- Ajouter `@commands.has_permissions(manage_messages=True)` et `@commands.guild_only()`.
- Remplacer la boucle de `delete()` par `await ctx.channel.purge(limit=nombre + 1)`.
- Gérer `commands.MissingPermissions` dans `on_command_error` pour répondre proprement.

#### Critères de validation
- [ ] Un membre sans « Gérer les messages » reçoit un refus et rien n'est supprimé.
- [ ] Un modérateur peut toujours utiliser la commande.

_Ticket : [`tickets/01-clear-utilisable-par-n-importe-qui.md`](tickets/01-clear-utilisable-par-n-importe-qui.md)_

### 02. Le bot peut être forcé à pinguer `@everyone`

**Priorité :** P0  
**Vérification :** 📖 Lecture du code  
**Où :** `commands.Bot(...)` (`bot.py:62`) ; vecteurs : `bot.py:420`, `bot.py:431`, `bot.py:1141`, `bot.py:1963`, `bot.py:2122`, `bot.py:3459`

#### Problème
Aucun `allowed_mentions` dans le projet. Le bot a la permission Administrateur, donc tout texte utilisateur qu'il renvoie peut contenir un `@everyone` / `@here` / `<@&rôle>` qui pingue réellement.

Vecteurs persistants (le texte est stocké puis renvoyé plus tard à d'autres personnes) :
- `--addInsult @everyone …` → `txt/insultes.txt`, renvoyé quand quelqu'un écrit « tg ».
- `--addBranlette jme @everyone …` → `txt/branlette.txt`, renvoyé sur « branle ».

Vecteurs directs : `/ask`, `--poll`, `--crypt` (le texte original est renvoyé).

Pas un vecteur : `/addquidenous`, car les questions sont affichées dans un embed, et Discord ne pingue jamais depuis un embed.

#### Preuve
Non testé sur un vrai serveur. Le ping effectif dépend de la permission « Mentionner @everyone », incluse dans Administrateur.

#### Correctif proposé
- Passer `allowed_mentions=discord.AllowedMentions(everyone=False, roles=False, users=True, replied_user=True)` à `commands.Bot(...)`.
- Pour les rares messages qui doivent vraiment pinguer (`bot.py:876`, niveau 100 dans le cog), passer explicitement `allowed_mentions=discord.AllowedMentions(everyone=True)` sur ce `send` uniquement.
- Nettoyer les entrées déjà présentes dans `insultes.txt` / `branlette.txt` qui contiendraient `@everyone`/`@here`.

#### Critères de validation
- [ ] `--addInsult @everyone test` puis « tg » n'envoie aucune notification.
- [ ] Les pings voulus (feur, niveau 100) fonctionnent toujours.

_Ticket : [`tickets/02-le-bot-peut-etre-force-a-pinguer-everyone.md`](tickets/02-le-bot-peut-etre-force-a-pinguer-everyone.md)_

### 03. `--dhcp` peut saturer la RAM du LXC

**Priorité :** P0  
**Vérification :** ✅ Exécuté  
**Où :** `bot.py:3562-3622`

#### Problème
`[str(ip) for ip in network]` matérialise toutes les adresses du réseau demandé, sans limite de taille.

#### Preuve
Pic de RAM mesuré sur la vraie commande (attente de 45 s court-circuitée) :
- `10.0.0.0/10` (4 M d'adresses) : +290 Mo, boucle bloquée 4,7 s.
- `10.0.0.0/8` (16 M) : **+1,16 Go**, 19 s.
- `0.0.0.0/0` : 256 × `/8`, soit environ 300 Go (extrapolé, non testé). Le processus sera tué par l'OOM killer.

Pas de cooldown sur cette commande.

#### Correctif proposé
- Refuser les réseaux plus grands qu'un `/22`, par exemple, via `network.num_addresses`.
- Ne générer que le nombre d'IP nécessaire (`itertools.islice(network.hosts(), len(users) + 1)`).
- Gérer le cas « plus de participants que d'IP » (`IndexError` actuel sur `ips.pop(0)`).
- Ajouter un cooldown.

#### Critères de validation
- [ ] `--dhcp 0.0.0.0/0` est refusé immédiatement.
- [ ] Un `/24` fonctionne comme avant.

_Ticket : [`tickets/03-dhcp-peut-saturer-la-ram-du-lxc.md`](tickets/03-dhcp-peut-saturer-la-ram-du-lxc.md)_

### 04. `--prime` charge 97 Mo en mémoire à chaque appel et peut se bloquer définitivement

**Priorité :** P0  
**Vérification :** ✅ Exécuté  
**Où :** `bot.py:2199-2246`, `txt/primes.txt`

#### Problème
`--prime` lit tout `txt/primes.txt` (97 Mo, 10,4 M de lignes) avec `readlines()`, **uniquement pour récupérer le dernier nombre**. Le fichier envoyé à l'utilisateur est un autre fichier : `txt/prime.txt` (7 Mo).

En plus, le `return` sur la branche « Primo no » ne décrémente pas `nbprime`. Après 3 passages par cette branche, la commande répond « je suis occupé » jusqu'au redémarrage.

`logger.info(nb, biggest, n_max)` est aussi un mauvais appel du logger : il affiche une erreur de formatage sur stderr.

#### Preuve
Mesuré : **+720 Mo de RAM** et 3,2 s de gel par appel. Jusqu'à 3 appels simultanés sont autorisés, soit environ 2,2 Go (extrapolé). Le compteur bloqué vient de la lecture du code.

#### Correctif proposé
- Remplacer la lecture par une constante `BIGGEST_PRIME = 187465331`, et retirer `txt/primes.txt` du dépôt (à faire upstream, voir ticket hygiène).
- Utiliser `try/finally` autour du compteur, ou un `asyncio.Semaphore(1)`.
- Corriger le `logger.info`.

#### Critères de validation
- [ ] `--prime 5` n'augmente plus la RAM de façon notable.
- [ ] Après 10 appels sur la branche « Primo no », la commande répond toujours normalement.

_Ticket : [`tickets/04-prime-charge-97-mo-en-memoire-a-chaque-appel-et-pe.md`](tickets/04-prime-charge-97-mo-en-memoire-a-chaque-appel-et-pe.md)_

### 05. `--calcul` : un exposant énorme gèle le bot pendant des minutes

**Priorité :** P0  
**Vérification :** ✅ Exécuté  
**Où :** `bot.py:2042-2117`

#### Problème
`nb1 ** nb2` est calculé sans borne, de façon synchrone, dans la boucle asyncio. Pendant le calcul, le bot ne répond plus nulle part. Le cooldown (3 appels / 5 s par utilisateur) ne protège pas, car un seul appel suffit.

#### Preuve
Mesuré sur la vraie commande :
- `--calcul 9^9999999` : 6,6 s de gel.
- `--calcul 9^99999999` : **223 s de gel**, +166 Mo.

Le résultat n'est même pas affichable : `str()` lève `ValueError` au-delà de 4300 chiffres.

#### Correctif proposé
- Refuser si `nb2 * log10(nb1) > 4000` (le résultat dépasserait la limite d'affichage Discord).
- Borner aussi `*` par la taille du résultat.
- Remplacer `facto()` récursif par `math.factorial` (garder la borne 806 : 806! fait 1995 chiffres, juste sous la limite Discord).

#### Critères de validation
- [ ] `--calcul 9^99999999` répond immédiatement par un refus.
- [ ] `--calcul 2^10` répond `1024`.

_Ticket : [`tickets/05-calcul-un-exposant-enorme-gele-le-bot-pendant-des.md`](tickets/05-calcul-un-exposant-enorme-gele-le-bot-pendant-des.md)_

### 06. `--isPrime` / `/isprime` : algorithme en O(√n) avec 3 `print` par tour

**Priorité :** P0  
**Vérification :** ✅ Exécuté  
**Où :** `fonctions.py:23-39`, `bot.py:2251`, `bot.py:2265`

#### Problème
`is_prime` teste les diviseurs jusqu'à √n et fait 3 `print` à chaque itération, de façon synchrone. La borne acceptée est 1e29. `/isprime` n'a aucun cooldown.

#### Preuve
Mesuré : un premier proche de 1e14 → 3,2 s de gel et 5 millions de lignes imprimées. Pour un premier proche de 1e28, c'est 10⁷ fois plus d'itérations, soit de l'ordre d'un an (extrapolé). Les `print` remplissent aussi les logs (journald / fichier) du LXC.

#### Correctif proposé
- Supprimer les `print`.
- Utiliser `sympy.isprime`, ou un Miller-Rabin déterministe (exact jusqu'à 3,3·10²⁴ avec les 13 premiers nombres premiers comme bases ; abaisser la borne à 1e24 dans ce cas).
- Ajouter un cooldown à `/isprime`.

#### Critères de validation
- [ ] `--isPrime 99999999999999999989` répond en moins de 10 ms.
- [ ] Plus aucune sortie `print` dans les logs.

_Ticket : [`tickets/06-isprime-isprime-algorithme-en-o-n-avec-3-print-par.md`](tickets/06-isprime-isprime-algorithme-en-o-n-avec-3-print-par.md)_

### 07. Flags du CTF en clair dans le dépôt public

**Priorité :** P0  
**Vérification :** ✅ Exécuté  
**Où :** `bot.py:2686-2705`

#### Problème
`FLAG` et `FLAG2` (`CYBN{…}`) sont écrits en dur dans le code d'un dépôt public : il suffit de lire GitHub pour valider le challenge.

De plus, `/flag` appelle `send_message` deux fois dès qu'un joueur y a droit (la branche `else` s'exécute toujours dans ce cas).

#### Preuve
Appel réel de `/flag` : le flag est bien envoyé, puis la 2ᵉ réponse lève `InteractionResponded`. Testé pour un joueur à 3 victoires et pour un joueur avec un match nul.

#### Correctif proposé
- Lire les flags depuis `.env` (`os.getenv("CTF_FLAG")`).
- **Changer les flags** côté CTF : les actuels sont publics depuis longtemps.
- Restructurer `/flag` en `if/elif/else` avec une seule réponse (les deux flags dans un seul message si besoin).

#### Critères de validation
- [ ] `grep CYBN bot.py` ne renvoie rien.
- [ ] `/flag` ne lève plus d'exception.

_Ticket : [`tickets/07-flags-du-ctf-en-clair-dans-le-depot-public.md`](tickets/07-flags-du-ctf-en-clair-dans-le-depot-public.md)_

### 08. Tokens Discord présents dans l'historique git

**Priorité :** P0  
**Vérification :** ✅ Exécuté  
**Où :** Historique git (commits de 2019 à 2021-07)

#### Problème
13 tokens distincts du bot (préfixe `NjUzNTYz…`, émis le 2019-12-09) sont lisibles dans l'historique public. `txt/admin.txt` et `txt/names.txt` ont aussi été commités en 2020, puis retirés.

#### Preuve
Trouvés par recherche dans `git log --all -p`. **Révocation non vérifiable de mon côté** : il faut le portail développeur Discord.

#### Correctif proposé
- Vérifier dans le portail développeur que le token actuel n'est pas l'un d'eux. En cas de doute, cliquer sur « Reset Token ».
- Ne pas réécrire l'historique pour ça : un token révoqué est inoffensif, et la réécriture casserait tous les forks.

#### Critères de validation
- [ ] Le propriétaire du bot confirme que le token actuel a été généré après 2021-07.

_Ticket : [`tickets/08-tokens-discord-presents-dans-l-historique-git.md`](tickets/08-tokens-discord-presents-dans-l-historique-git.md)_

### 09. Écritures de fichiers non atomiques : un crash peut effacer les données Pokémon

**Priorité :** P1  
**Vérification :** 📖 Lecture du code  
**Où :** `cogs/pokemon_starter.py:165-197`, `bot.py:405`, `bot.py:3270`

#### Problème
`save_pokemon_data` réécrit tout `data/pokemon_starters.json` directement (`open(..., "w")`). Si le processus est tué pendant l'écriture (OOM dans le LXC, redémarrage…), le JSON est tronqué. Au démarrage suivant, `load_pokemon_data` attrape `JSONDecodeError` et **renvoie `{}`**. Le premier message reçu sauvegarde alors ce `{}` : toutes les données Pokémon sont perdues sans alerte.

Même schéma pour `dico.txt` (réécrit à chaque mot nouveau), `leaderboard.txt`, `pve.txt` et `server_names.txt`.

C'est le lien direct avec le risque LXC : les tickets mémoire (dhcp, prime, calcul) peuvent provoquer cette perte.

#### Preuve
Déduit de la lecture du code, non reproduit.

#### Correctif proposé
- Écriture atomique : écrire dans `fichier.tmp`, puis `os.replace(tmp, fichier)`.
- Sur `JSONDecodeError`, ne **pas** repartir de `{}` : renommer le fichier corrompu en `.corrupt-<date>` et refuser de sauvegarder.
- Ajouter une sauvegarde quotidienne de `data/`.

#### Critères de validation
- [ ] Tuer le processus pendant une sauvegarde ne laisse jamais de JSON tronqué.

_Ticket : [`tickets/09-ecritures-de-fichiers-non-atomiques-un-crash-peut.md`](tickets/09-ecritures-de-fichiers-non-atomiques-un-crash-peut.md)_

### 10. Appels HTTP bloquants dans la boucle asyncio

**Priorité :** P1  
**Vérification :** 📖 Lecture du code  
**Où :** `bot.py` (kanye, skin, `/chat` l.3541, `/activity`), `cogs/pokemon_starter.py:276`, `:288`, `:851`, `:889`

#### Problème
Tous les appels réseau utilisent `requests.get`, qui est synchrone : chaque appel gèle tout le bot le temps de la réponse. `/chat` n'a même pas de `timeout`, donc un serveur lent peut bloquer le bot indéfiniment.

#### Preuve
Déduit de la lecture du code.

#### Correctif proposé
- Passer à `aiohttp` (déjà installé avec discord.py), ou envelopper avec `await asyncio.to_thread(requests.get, ...)`.
- Mettre un `timeout` partout.

#### Critères de validation
- [ ] Aucun `requests.get` direct dans une coroutine.
- [ ] `/chat` a un timeout.

_Ticket : [`tickets/10-appels-http-bloquants-dans-la-boucle-asyncio.md`](tickets/10-appels-http-bloquants-dans-la-boucle-asyncio.md)_

### 11. Permissions excessives : Administrateur demandé à l'invitation + `Intents.all()`

**Priorité :** P1  
**Vérification :** 📖 Lecture du code  
**Où :** `README.md`, `bot.py:49`, `bot.py:59-61`, commande `/invite` (`bot.py:2561`)

#### Problème
Le lien d'invitation demande `permissions=8` (Administrateur). Toute faille du bot (cf. `--clear`, `@everyone`) s'exécute donc avec les pleins pouvoirs. `Intents.all()` active aussi les intents privilégiés, dont la présence, qui n'est pas utilisée.

`client = discord.Client(intents=intents)` (l.61) est créé mais jamais utilisé.

#### Preuve
Déduit de la lecture du code.

#### Correctif proposé
- Calculer les permissions réellement nécessaires (envoyer des messages, embeds, fichiers, réactions, gérer les messages, threads, pseudo, vocal) et régénérer le lien.
- Remplacer `Intents.all()` par `Intents.default()` + `message_content` + `members`.
- Supprimer `client`.
- Sur les serveurs existants, retirer le rôle admin du bot.

#### Critères de validation
- [ ] Le lien d'invitation ne contient plus `permissions=8`.
- [ ] Le bot fonctionne sans le rôle Administrateur.

_Ticket : [`tickets/11-permissions-excessives-administrateur-demande-a-l.md`](tickets/11-permissions-excessives-administrateur-demande-a-l.md)_

### 12. Contenu des conversations et données des membres publiés sur GitHub

**Priorité :** P1  
**Vérification :** 📖 Lecture du code  
**Où :** `bot.py:376-406`, `txt/dico.txt`, `txt/leaderboard.txt`, `txt/pve.txt`, `txt/server_names.txt`, `data/`

#### Problème
Le bot lit tous les messages de tous les serveurs et ajoute chaque mot nouveau à `txt/dico.txt`. Ce fichier est ensuite commité sur le dépôt public (commits « prod »). Les logs associent pseudo + serveur + mot. Les leaderboards (ID Discord + pseudo) sont aussi publics.

#### Preuve
Constaté dans l'historique : `dico.txt` passe de 33 912 lignes (commit `29e4dd1`, mai 2026) à 39 779 lignes aujourd'hui, via les commits « prod » de l'auteur « maybe push in prod ».

#### Correctif proposé
- Ne plus versionner les fichiers alimentés en production (voir ticket hygiène).
- Prévenir les serveurs que leurs messages alimentent un dictionnaire, ou désactiver la collecte.
- Arrêter de logger le mot avec le pseudo.

#### Critères de validation
- [ ] Plus aucun fichier modifié par le bot en prod n'est suivi par git.

_Ticket : [`tickets/12-contenu-des-conversations-et-donnees-des-membres-p.md`](tickets/12-contenu-des-conversations-et-donnees-des-membres-p.md)_

### 13. Ajouts libres dans les fichiers texte, sans limite ni modération

**Priorité :** P1  
**Vérification :** 📖 Lecture du code  
**Où :** `bot.py:420-443` (`--addInsult`, `--addBranlette`), `bot.py:4019` (`/addquidenous`)

#### Problème
N'importe qui peut ajouter du texte arbitraire (jusqu'à 2000 caractères par message, 6000 pour une option de slash) dans des fichiers que le bot renvoie ensuite à tout le monde. Il n'y a ni limite de taille, ni cooldown, ni possibilité de retirer une entrée sans éditer le fichier à la main. Ces fichiers sont aussi partagés entre **tous les serveurs** : un serveur peut injecter du contenu qui s'affichera sur un autre.

Effets : spam, contenu choquant/haineux propagé, croissance du disque.

#### Preuve
Déduit de la lecture du code.

#### Correctif proposé
- Longueur max (ex. 200 caractères) et cooldown.
- Commande de suppression réservée aux admins.
- Optionnel : stockage par serveur.

#### Critères de validation
- [ ] Une entrée de plus de 200 caractères est refusée.
- [ ] Un admin peut supprimer une entrée.

_Ticket : [`tickets/13-ajouts-libres-dans-les-fichiers-texte-sans-limite.md`](tickets/13-ajouts-libres-dans-les-fichiers-texte-sans-limite.md)_

### 14. `/unban` et `/immature` retirent aussi le salon suivant de la liste

**Priorité :** P2  
**Vérification :** ✅ Exécuté  
**Où :** `bot.py:2484-2495` (`/unban`), `bot.py:2547-2558` (`/immature`)

#### Problème
La boucle fait `bansLines.remove(id)` pendant qu'elle parcourt `bansLines`. L'élément suivant est sauté, donc jamais réécrit dans le fichier.

#### Preuve
Appel réel de `/unban` : `bans.txt` = `111 222 333 444`, unban du salon 222 → le fichier contient `111 444`. Le salon 333 est débanni sans que personne ne l'ait demandé. `/immature` utilise exactement le même code (non exécuté).

#### Correctif proposé
Écrire la liste filtrée : `f.writelines(l for l in lines if l != chanID)`.

#### Critères de validation
- [ ] Unban de 222 parmi `111 222 333 444` → `111 333 444`.

_Ticket : [`tickets/14-unban-et-immature-retirent-aussi-le-salon-suivant.md`](tickets/14-unban-et-immature-retirent-aussi-le-salon-suivant.md)_

### 15. `--repeat` plante systématiquement

**Priorité :** P2  
**Vérification :** ✅ Exécuté  
**Où :** `bot.py:1944-1950`

#### Problème
`ctx.channel.history()` renvoie un générateur async. Le parcourir avec un `for` classique lève `TypeError`, avant même l'envoi du message.

#### Preuve
Appel réel avec discord.py 2.7.1 : `TypeError: 'async_generator' object is not iterable`, `send` jamais appelé. (Yanis indique que la commande marche chez lui : à vérifier si la prod tourne bien sur ce code.)

#### Correctif proposé
Remplacer par `await ctx.message.delete()`, puis `await ctx.send(...)`.

#### Critères de validation
- [ ] `--repeat salut` supprime le message d'origine et renvoie « salut ».

_Ticket : [`tickets/15-repeat-plante-systematiquement.md`](tickets/15-repeat-plante-systematiquement.md)_

### 16. Erreur Pokémon du jour : le message d'erreur n'est jamais envoyé

**Priorité :** P2  
**Vérification :** 📖 Lecture du code  
**Où :** `bot.py:835-845`

#### Problème
Dans le `except`, la première instruction est `await channel.send()` sans contenu. L'API Discord refuse les messages vides (erreur 50006). L'exception est donc relancée depuis le `except`, et les deux messages d'erreur suivants ne partent jamais.

#### Preuve
Déduit de la lecture du code et du comportement de l'API (non testé contre Discord).

#### Correctif proposé
Supprimer `await channel.send()` et le doublon du message.
Optionnel : ne pas afficher `str(e)` aux utilisateurs.

#### Critères de validation
- [ ] Quand le cache Pokémon est vide, l'image d'erreur et le message s'affichent.

_Ticket : [`tickets/16-erreur-pokemon-du-jour-le-message-d-erreur-n-est-j.md`](tickets/16-erreur-pokemon-du-jour-le-message-d-erreur-n-est-j.md)_

### 17. La taille « du jour » change à chaque redémarrage du bot

**Priorité :** P2  
**Vérification :** ✅ Exécuté  
**Où :** `bot.py:694`

#### Problème
`hash((user.id, "YYYY-MM-DD"))` contient une chaîne. Or Python randomise le hash des chaînes à chaque lancement (`PYTHONHASHSEED`). La valeur « du jour » n'est donc stable que jusqu'au prochain redémarrage. Les stats de `/sexestats` restent cohérentes, mais l'affichage ne l'est pas.

#### Preuve
Même utilisateur, même date, 3 lancements de Python : tailles 12, 26 et 4.

#### Correctif proposé
Utiliser une graine stable : `int(hashlib.sha256(f"{user.id}:{date}".encode()).hexdigest(), 16)` et `random.Random(seed)` (au lieu de reseeder le `random` global).

#### Critères de validation
- [ ] Deux lancements du bot donnent la même taille pour le même utilisateur le même jour.

_Ticket : [`tickets/17-la-taille-du-jour-change-a-chaque-redemarrage-du-b.md`](tickets/17-la-taille-du-jour-change-a-chaque-redemarrage-du-b.md)_

### 18. Crashs en messages privés (DM)

**Priorité :** P2  
**Vérification :** 📖 Lecture du code  
**Où :** `bot.py:399` et toutes les lignes `message.guild.name` de `on_message`, `--appel`, `--rename`, `--clear`, `/combat` (cog l.1139)

#### Problème
En DM, `message.guild` vaut `None`. `on_message` utilise `message.guild.name` dans les logs dès qu'un mot nouveau est vu, donc `AttributeError`. Les commandes qui utilisent `guild_permissions` ou `interaction.guild.id` plantent aussi.

#### Preuve
Déduit de la lecture du code.

#### Correctif proposé
- Ignorer les DM en tête de `on_message` (ou utiliser `guild_name = message.guild.name if message.guild else "DM"`).
- Ajouter `@commands.guild_only()` / `@app_commands.guild_only()` sur les commandes de serveur.

#### Critères de validation
- [ ] Envoyer un DM au bot ne produit aucune trace d'exception.

_Ticket : [`tickets/18-crashs-en-messages-prives-dm.md`](tickets/18-crashs-en-messages-prives-dm.md)_

### 19. Pokémon : cooldowns de combat et d'XP contournables

**Priorité :** P2  
**Vérification :** 📖 Lecture du code  
**Où :** `cogs/pokemon_starter.py:1158-1189` (combat), `:741-756` (XP)

#### Problème
- **Combat** : `last_combat` n'est écrit qu'après l'acceptation (jusqu'à 60 s plus tard), et seul le cooldown du lanceur est vérifié. On peut donc lancer plusieurs défis en parallèle, et défier quelqu'un qui est lui-même en cooldown.
- **XP** : `entry["last_time"]` n'est mis à jour qu'après `evolve()`, qui fait plusieurs `sleep` (~4 s). Pendant l'animation, chaque message du joueur rapporte de l'XP.

#### Preuve
Déduit de la lecture du code.

#### Correctif proposé
- Combat : vérifier le cooldown des deux joueurs, et poser `last_combat` (ou un verrou « en combat ») dès l'envoi du défi.
- XP : écrire `entry["last_time"] = now` avant le moindre `await`.

#### Critères de validation
- [ ] Deux `/combat` lancés coup sur coup : le second est refusé.

_Ticket : [`tickets/19-pokemon-cooldowns-de-combat-et-d-xp-contournables.md`](tickets/19-pokemon-cooldowns-de-combat-et-d-xp-contournables.md)_

### 20. Mèmes `--master` / `--presentation` : fichier partagé entre utilisateurs

**Priorité :** P2  
**Vérification :** 📖 Lecture du code  
**Où :** `bot.py:2388`, `bot.py:2431`

#### Problème
L'image générée est écrite dans `images/mastermeme.jpg` / `images/presentationmeme.png` (fichiers suivis par git), puis relue pour l'envoi. Si deux personnes lancent la commande en même temps, l'une peut recevoir l'image de l'autre. Ça modifie aussi des fichiers versionnés en prod.

#### Preuve
Déduit de la lecture du code.

#### Correctif proposé
Générer dans un `io.BytesIO` et envoyer `discord.File(buf, filename=...)`, comme `/sexestats` le fait déjà (l.1935). Retirer les deux fichiers du dépôt.

#### Critères de validation
- [ ] Aucun fichier n'est écrit sur disque par ces commandes.

_Ticket : [`tickets/20-memes-master-presentation-fichier-partage-entre-ut.md`](tickets/20-memes-master-presentation-fichier-partage-entre-ut.md)_

### 21. Leaderboards : format texte fragile et recherche par sous-chaîne

**Priorité :** P2  
**Vérification :** 📖 Lecture du code  
**Où :** `bot.py:3269-3370`, `/flag`

#### Problème
Lignes `id-victoires-défaites-ratio-nom`. `getScoreLeaderBoard` et `getPlaceLeaderbord` cherchent avec `str(id) in ligne`, une recherche par sous-chaîne. Un nom d'utilisateur (chiffres autorisés) égal à l'ID d'un autre joueur peut faire renvoyer la mauvaise ligne, ce qui compte pour `/flag`. Pas d'écriture atomique non plus (cf. ticket écritures).

#### Preuve
Déduit de la lecture du code. Les noms d'utilisateur Discord actuels n'acceptent pas `-`, donc le décalage de colonnes est peu probable (correction de ma première analyse).

#### Correctif proposé
Passer à un JSON `{user_id: {wins, losses, name}}`, avec écriture atomique.

#### Critères de validation
- [ ] Le score est retrouvé uniquement par égalité exacte de l'ID.

_Ticket : [`tickets/21-leaderboards-format-texte-fragile-et-recherche-par.md`](tickets/21-leaderboards-format-texte-fragile-et-recherche-par.md)_

### 22. Dépendances : Pillow vulnérable, Python ≥ 3.12 requis, numpy non épinglé

**Priorité :** P2  
**Vérification :** ✅ Exécuté  
**Où :** `requirements.txt`, `README.md`

#### Problème
- `pip-audit` signale **13 vulnérabilités** dans `pillow 12.2.0`, toutes corrigées en 12.3.0. L'exposition est faible : le bot n'ouvre que ses propres images et des images de GitHub/Wikimedia, pas des fichiers fournis par les utilisateurs.
- `Tyradex` (via `pratik`) utilise une syntaxe f-string réservée à **Python ≥ 3.12** : l'import échoue en 3.11. Le README annonce Python 3.10.2.
- `numpy` n'a pas de version épinglée.

#### Preuve
`pip-audit -r requirements.txt` exécuté. L'échec d'import a été constaté en Python 3.11.15.

#### Correctif proposé
- `pillow~=12.3.0`, épingler `numpy`.
- Mettre à jour le README (Python 3.12+).

#### Critères de validation
- [ ] `pip-audit -r requirements.txt` ne remonte rien.
- [ ] Le README indique la bonne version de Python.

_Ticket : [`tickets/22-dependances-pillow-vulnerable-python-3-12-requis-n.md`](tickets/22-dependances-pillow-vulnerable-python-3-12-requis-n.md)_

### 23. Petits bugs et nettoyages divers

**Priorité :** P2  
**Vérification :** 📖 Lecture du code  
**Où :** Voir la liste

#### Problème
- `--randomWord N` : pas de borne ni de cooldown. Mesuré : N=3 000 000 → 1,2 s de gel, message de 25 M caractères refusé par Discord. Mineur.
- `logger.info("…:", mot)` (`bot.py:442`) et `logger.info(nb, biggest, n_max)` : mauvais usage du logger, ce qui affiche des erreurs de formatage.
- `facto()` récursif (`fonctions.py:100`) : remplacer par `math.factorial`.
- « Qui d'entre nous » : un menu de sélection Discord est limité à 25 options. Au-delà de 25 joueurs, la création du vote échoue.
- `/mature` et `/immature` sont définies avec les noms Python `ban` / `unban`, ce qui écrase les fonctions précédentes. Pas de bug fonctionnel (les commandes sont enregistrées à la décoration), mais c'est trompeur.
- `random.seed(seed)` puis `random.seed(None)` sur le générateur global (`bot.py:695`) : utiliser `random.Random(seed)`.

#### Preuve
`randomWord` exécuté. Le reste vient de la lecture du code.

#### Correctif proposé
Corriger au cas par cas.

#### Critères de validation
- [ ] Chaque point de la liste est corrigé ou explicitement abandonné.

_Ticket : [`tickets/23-petits-bugs-et-nettoyages-divers.md`](tickets/23-petits-bugs-et-nettoyages-divers.md)_

### 24. Hygiène du dépôt (à traiter sur le dépôt original)

**Priorité :** P2  
**Vérification :** ✅ Exécuté  
**Où :** Dépôt `NozyZy/Le-ptit-bot`

#### Problème
- `txt/primes.txt` fait 97 Mo, proche de la limite GitHub de 100 Mo. 7 versions de 83 à 97 Mo sont dans l'historique (pack de 67 Mo). Inutile, cf. ticket `--prime`.
- Les fichiers d'état modifiés en prod sont versionnés : `tg.txt`, `onecops_counter.txt`, `leaderboard.txt`, `pve.txt`, `bans.txt`, `mature.txt`, `dico.txt`, `server_names.txt`, mèmes générés.
- 3174 fichiers d'un `venv/` ont été commités par le passé.
- `.gitignore` de 961 lignes (template générique).
- Bits exécutables sur des images, le README, la licence (déjà corrigé sur la branche `claude/cleanup-security-review` du fork).

#### Preuve
Constaté avec `git rev-list --objects`, `git count-objects` et `git ls-files -s`.

#### Correctif proposé
- Déplacer l'état dans `data/` (déjà ignoré), et fournir des fichiers `*.example` pour l'initialisation.
- `git rm --cached` des fichiers d'état et de `primes.txt`.
- Ne pas réécrire l'historique (ça casserait les forks), sauf décision explicite du propriétaire.
- **À faire upstream** : si on le fait seulement dans le fork, chaque synchronisation créera des conflits.

#### Critères de validation
- [ ] Après une semaine de prod, `git status` reste propre sur le serveur.

_Ticket : [`tickets/24-hygiene-du-depot-a-traiter-sur-le-depot-original.md`](tickets/24-hygiene-du-depot-a-traiter-sur-le-depot-original.md)_

## 6. Corrections par rapport à la première revue

Une première revue a été rendue plus tôt dans la session. Plusieurs points étaient imprécis ou faux :

- **« Testé »** était utilisé pour des tests qui recopiaient la logique au lieu d'appeler le vrai code (`/unban`, `--calcul`),
  et pour `--randomWord` / `--dhcp`, qui n'avaient pas été exécutés du tout. Tout a été refait sur le vrai code (section 1).
- **`/addquidenous` n'est pas un vecteur `@everyone`** : les questions sont affichées dans un embed, qui ne pingue jamais.
- **`--randomWord` était surestimé** : c'est un point mineur (1,2 s pour 3 M de mots, message refusé par Discord).
- **Leaderboard** : les noms d'utilisateur Discord actuels n'acceptent pas `-`, donc le « décalage de colonnes » est peu probable.
  La recherche par sous-chaîne reste un problème.
- **`primes.txt` « se régénère en quelques secondes »** : affirmation non vérifiée, retirée.
  En réalité, il ne sert qu'à lire son dernier nombre (point 04).
- **Ajouts** : perte de données Pokémon (09), cooldowns contournables (19), taille non stable (17), message d'erreur Pokémon (16),
  fichiers de mèmes partagés (20), `/immature` touché par le même bug que `/unban` (14), Pillow vulnérable (22), Python ≥ 3.12 (22).

## 7. État du fork et du dépôt git

- Le fork `Skiepy/Le-ptit-bot` a été mis à jour en fast-forward sur `NozyZy/Le-ptit-bot` (27 commits de retard, aucun commit propre au fork).
- Côté GitHub, le fork était déjà propre : une seule branche `main`, aucune PR. **Les issues sont désactivées** (réglage par défaut des forks).
- Branche `claude/cleanup-security-review` : retire les bits exécutables parasites (22 fichiers, changement de mode uniquement).
- Branche `claude/audit-securite` (celle-ci) : ce rapport et les tickets.
- Le gros du ménage (point 24) doit se faire **sur le dépôt original**. Sinon le fork divergera et chaque synchronisation créera des conflits.

## 8. Ce qui n'a pas été vérifié

- Le comportement réel face à l'API Discord (ping `@everyone`, message vide refusé), faute de serveur de test.
- La révocation des anciens tokens (08) : il faut l'accès au portail développeur.
- La configuration réelle du LXC (RAM allouée, limites cgroup, swap) : les seuils exacts de plantage en dépendent.
- Le code réellement déployé en production. Yanis indique que `--repeat` fonctionne chez lui, alors que ce code plante avec discord.py 2.7.1.
  Si la prod tourne sur une autre version, certains points peuvent différer.
- `images/italian.py` : script hors ligne qui n'est pas exécuté par le bot, survolé seulement.
