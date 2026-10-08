---
layout: article
title: "Augmenter la limite inotify sur Debian et Ubuntu"
description: "Corriger l'erreur inotify watch limit reached sur Debian et Ubuntu : comprendre les limites d'inotify et les augmenter de façon temporaire ou permanente."
tags:
  - linux
  - debian
  - ubuntu
  - inotify
author: Pierre Chopinet
---

Les IDE, les serveurs de développement avec rechargement automatique ou un simple `tail -f` utilisent inotify pour savoir quand un fichier change. Sur un gros projet, ou avec beaucoup de conteneurs, on finit par atteindre les limites fixées par le noyau et par tomber sur des erreurs comme `ENOSPC: System limit for number of file watchers reached` ou `Too many open files`. Voyons comment vérifier ces limites et les augmenter sur Debian et Ubuntu.
<!--more-->

Dans cet article :
- Comment fonctionne inotify
- Les trois limites
- Vérifier les limites actuelles
- Augmenter les limites temporairement
- Rendre la modification permanente
- Combien de mémoire consomme un watch
- Trouver les processus qui utilisent inotify

Pré-requis : Debian ou Ubuntu (la procédure est la même sur les autres distributions) et un accès `sudo`.

## Comment fonctionne inotify

inotify (pour *inode notify*) est l'API du noyau Linux qui permet à un programme d'être prévenu quand un fichier ou un répertoire est créé, modifié, supprimé ou déplacé. Le programme crée d'abord une *instance* inotify, puis il y ajoute des *watches*, un par fichier ou répertoire à surveiller. La surveillance d'un répertoire n'est pas récursive : un outil qui surveille tout un projet pose un watch sur chacun de ses sous-répertoires.

Pour voir inotify en action, on peut utiliser `inotifywait`, du paquet `inotify-tools` :

```bash
sudo apt install inotify-tools
inotifywait -m -r projet
```

L'option `-m` laisse tourner la commande et `-r` surveille aussi les sous-répertoires. En modifiant deux fichiers du projet depuis un autre terminal, on obtient :

```
Setting up watches.  Beware: since -r was given, this may take a while!
Watches established.
projet/src/a/1/ OPEN f1.js
projet/src/a/1/ ATTRIB f1.js
projet/src/a/1/ CLOSE_WRITE,CLOSE f1.js
projet/src/b/2/ OPEN f2.js
projet/src/b/2/ MODIFY f2.js
projet/src/b/2/ CLOSE_WRITE,CLOSE f2.js
```

Le projet de test contient 112 répertoires et 200 fichiers : `inotifywait` y a posé 112 watches, un par répertoire, les fichiers étant couverts par le watch de leur répertoire. Sur un gros projet, ou avec un dossier `node_modules` qui peut contenir des milliers de sous-répertoires, le nombre de watches grimpe donc vite.

## Les trois limites

Le noyau limite l'utilisation d'inotify avec trois paramètres, que l'on trouve dans `/proc/sys/fs/inotify/`.

`max_user_watches` fixe le nombre maximum de watches par utilisateur, tous processus confondus. C'est la limite que l'on atteint le plus souvent. Au-delà, l'ajout d'un watch échoue avec l'erreur `ENOSPC` ("No space left on device"), même si le disque n'est pas plein.

Sa valeur par défaut dépend de la version du noyau. Jusqu'au noyau 5.10, elle valait 8192, ce qui est vite atteint. Depuis le noyau 5.11, elle est calculée au démarrage pour que les watches d'un utilisateur ne puissent pas occuper plus d'environ 1 % de la RAM, avec un minimum de 8192 et un maximum de 1 048 576. Ubuntu 22.04 (noyau 5.15) et les versions suivantes en profitent. Pour connaître la version de votre noyau, lancez `uname -r`.

`max_user_instances` fixe le nombre maximum d'instances par utilisateur, 128 par défaut. Chaque programme qui utilise inotify en ouvre au moins une. Au-delà, la création d'une instance échoue avec `EMFILE`, c'est-à-dire "Too many open files", la même erreur que lorsqu'un processus dépasse son nombre maximum de fichiers ouverts. On peut atteindre cette limite avec beaucoup de conteneurs lancés par le même utilisateur, souvent root. C'est un problème connu de kind, l'outil qui fait tourner des clusters Kubernetes dans Docker, dès que le cluster a plusieurs nœuds.

Enfin, `max_queued_events` fixe le nombre d'événements qui peuvent attendre d'être lus dans la file d'une instance, 16 384 par défaut. Au-delà, les nouveaux événements sont perdus et le programme reçoit un événement `IN_Q_OVERFLOW` : à lui de relire l'état des fichiers.

Chaque programme signale les erreurs `ENOSPC` et `EMFILE` à sa façon. Voici ce qu'affichent Node.js, `tail -f` et `inotifywait` quand il n'y a plus de watches disponibles :

```
Error: ENOSPC: System limit for number of file watchers reached, watch 'projet/src/e/3'
tail: inotify resources exhausted
Failed to watch projet; upper limit on inotify watches reached!
```

Et quand c'est le nombre d'instances qui est atteint :

```
Error: EMFILE: too many open files, watch 'projet'
tail: inotify cannot be used, reverting to polling: Too many open files
Couldn't initialize inotify: Too many open files
```

Comme on le voit avec `tail`, certains programmes se rabattent sur une scrutation régulière des fichiers (*polling*), moins réactive : par défaut, `tail -f` ne vérifie alors le fichier qu'une fois par seconde.

## Vérifier les limites actuelles

La commande `sysctl` affiche les trois valeurs d'un coup :

```bash
sysctl fs.inotify
```

Sur une machine de 16 Go de RAM avec un noyau récent, on obtient :

```
fs.inotify.max_queued_events = 16384
fs.inotify.max_user_instances = 128
fs.inotify.max_user_watches = 130054
```

On peut aussi lire directement les fichiers, par exemple avec `cat /proc/sys/fs/inotify/max_user_watches`. Nous verrons plus bas d'où viennent ces 130 054 watches.

## Augmenter les limites temporairement

Avant de toucher aux limites, il est souvent plus simple de ne pas surveiller les dossiers qui n'en ont pas besoin. La documentation de VS Code conseille par exemple d'ajouter les gros dossiers comme un `.venv` Python au réglage `files.watcherExclude` avant d'augmenter `max_user_watches`.

Si cela ne suffit pas, on modifie les valeurs avec `sysctl -w` (le `-w` est facultatif quand on donne une valeur) :

```bash
sudo sysctl fs.inotify.max_user_watches=524288
sudo sysctl fs.inotify.max_user_instances=512
```

Chaque commande affiche la nouvelle valeur :

```
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 512
```

Ce sont les valeurs recommandées par la documentation de VS Code pour les watches, et par celle de kind pour les watches et les instances.

Le noyau vérifie ces limites à chaque ajout de watch et à chaque création d'instance : les nouvelles valeurs s'appliquent tout de suite. Par contre, un programme qui a déjà rencontré l'erreur ne réessaiera pas forcément de lui-même, il vaut mieux le relancer. La valeur de `max_queued_events`, elle, est lue à la création de chaque instance : un changement ne concerne que les programmes lancés après.

Attention, ces modifications sont perdues au redémarrage. C'est pratique pour tester, mais pour les garder il faut passer par un fichier de configuration.

## Rendre la modification permanente

Au démarrage, les fichiers `.conf` du dossier `/etc/sysctl.d/` sont lus et appliqués. On crée donc un fichier pour nos réglages :

```bash
sudo nano /etc/sysctl.d/60-inotify.conf
```

Et on y ajoute ces lignes :

```
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 512
```

Le nom du fichier importe peu, du moment qu'il se termine par `.conf`. Je n'ai pas mis `max_queued_events` : la valeur par défaut suffit tant qu'un programme ne signale pas de débordement de sa file d'événements.

Pour appliquer le fichier sans redémarrer :

```bash
sudo sysctl -p /etc/sysctl.d/60-inotify.conf
```

```
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 512
```

La commande `sudo sysctl --system` fait la même chose avec tous les fichiers de configuration (`/etc/sysctl.d/`, `/usr/lib/sysctl.d/`, `/etc/sysctl.conf`...), comme au démarrage. Il ne reste plus qu'à vérifier avec `sysctl fs.inotify` :

```
fs.inotify.max_queued_events = 16384
fs.inotify.max_user_instances = 512
fs.inotify.max_user_watches = 524288
```

## Combien de mémoire consomme un watch

Un watch n'est alloué qu'au moment où un programme le demande : augmenter la limite ne réserve aucune mémoire. Elle fixe seulement un plafond.

Pour calculer la valeur par défaut, le noyau estime le coût d'un watch à la taille de la structure qui le décrit, plus deux fois la taille d'un inode. En effet, tant qu'un fichier est surveillé, son inode reste en mémoire, et le noyau double la taille de l'inode générique pour couvrir la partie propre au système de fichiers. Sur un noyau 6.18 en x86-64, ces structures font respectivement 80 et 608 octets, soit 80 + 2 × 608 = 1 296 octets par watch. Sur la machine de 16 Go vue plus haut, 1 % de la RAM représente environ 168 Mo, et 168 Mo divisés par 1 296 octets donnent environ 130 000 watches : c'est bien la limite par défaut que nous avions obtenue (130 054).

Dans le pire des cas, si les 524 288 watches autorisés sont tous posés, ils occupent donc 524 288 × 1 296 octets, un peu moins de 700 Mo de mémoire noyau. C'est l'ordre de grandeur à garder en tête avant de monter la limite à plusieurs millions sur une machine qui a peu de RAM.

## Trouver les processus qui utilisent inotify

Quand la limite est atteinte, il faut savoir qui consomme les watches. Chaque instance inotify apparaît comme un descripteur de fichier `anon_inode:inotify` dans `/proc/PID/fd/`, et le fichier `/proc/PID/fdinfo/N` correspondant contient une ligne par watch :

```
inotify wd:1 ino:925a7 sdev:fe00000 mask:2 ignored_mask:0 fhandle-bytes:8 fhandle-type:1 f_handle:a72509008489f91a
```

Pour compter les instances ouvertes, puis le nombre total de watches :

```bash
find /proc/*/fd -lname 'anon_inode:inotify' 2>/dev/null | wc -l
cat /proc/*/fdinfo/* 2>/dev/null | grep -c '^inotify'
```

Attention à ne pas confondre les deux : une seule instance peut porter des milliers de watches. C'est `max_user_watches` qui limite les watches et `max_user_instances` les instances.

Pour avoir le détail par processus, voici un petit script :

```bash
#!/usr/bin/env bash
# Affiche, pour chaque processus qui utilise inotify, son nombre de watches et d'instances

printf '%8s %9s  %s\n' WATCHES INSTANCES 'PID COMMANDE'
find /proc/[0-9]*/fd -lname 'anon_inode:inotify' 2>/dev/null |
while IFS=/ read -r _ _ pid _ fd; do
    watches=$(grep -c '^inotify' "/proc/$pid/fdinfo/$fd")
    echo "$pid $(tr ' ' '_' < "/proc/$pid/comm") $watches"
done 2>/dev/null |
awk '{ inst[$1" "$2]++; w[$1" "$2] += $3 }
     END { for (p in w) printf "%8d %9d  %s\n", w[p], inst[p], p }' | sort -rn
```

Avec `inotifywait`, un script Node.js et un `tail -f` lancés sur le projet de test, on obtient :

```
 WATCHES INSTANCES  PID COMMANDE
     112         1  8052 node
     112         1  8050 inotifywait
       1         1  8054 tail
```

`inotifywait` et le script Node.js surveillent chacun les 112 répertoires du projet, avec une seule instance. `tail -f` n'a besoin que d'un watch, sur le fichier suivi.

Lancé sans `sudo`, le script ne voit que vos propres processus, ce qui suffit puisque les limites sont comptées par utilisateur. Avec `sudo`, il liste les processus de tous les utilisateurs.

## Voir aussi

- [free : surveiller et comprendre l'utilisation mémoire sous Linux]({% post_url 2026-03-16-Surveiller-la-memoire-avec-free %})
- [df/du : surveiller et analyser l'espace disque sous Linux]({% post_url 2026-03-01-Surveiller-espace-disque-avec-df-et-du %})
- [man inotify](https://man7.org/linux/man-pages/man7/inotify.7.html)
- [VS Code sous Linux](https://code.visualstudio.com/docs/setup/linux), dont la section sur l'erreur ENOSPC du *file watcher*
- [Problèmes connus de kind](https://kind.sigs.k8s.io/docs/user/known-issues/), section *Pod errors due to "too many open files"*
