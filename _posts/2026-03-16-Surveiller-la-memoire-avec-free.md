---
layout: article
title: "free : surveiller et comprendre l'utilisation mémoire sous Linux"
description: "Surveiller la mémoire sous Linux avec free : lire la sortie, comprendre buff/cache et available, options utiles et aller plus loin avec /proc/meminfo."
author: Pierre Chopinet
tags:
  - linux
  - free
  - cli
  - shell
  - bash
  - outils
  - sysadmin
  - memoire
---

Quand un serveur rame ou qu'un processus se fait tuer par l'OOM killer, la première chose à regarder est la mémoire. La commande `free` donne en une seconde l'état de la RAM et du swap, encore faut-il savoir lire ses colonnes : la plupart des fausses alertes viennent d'une mauvaise lecture de la colonne `free`.
<!--more-->

Dans cet article :
- Lire la sortie de free
- Pourquoi la colonne free est presque toujours basse
- Les options utiles
- Trouver les processus qui consomment la mémoire
- Le swap
- Vider le cache, une fausse bonne idée
- Aller plus loin avec /proc/meminfo

## Lire la sortie de free

On lance presque toujours `free` avec l'option `-h`, pour avoir des tailles lisibles :

```bash
free -h
```

Voici ce que donne la commande sur une machine de 16 Go qui fait tourner un processus gourmand et qui vient de lire un gros fichier :

```
               total        used        free      shared  buff/cache   available
Mem:            15Gi       5.7Gi       2.7Gi        13Mi       7.7Gi        10Gi
Swap:             0B          0B          0B
```

La colonne `free` indique seulement 2,7 Gi de libre, alors que la machine a en réalité 10 Gi de disponible pour lancer de nouvelles applications. Le chiffre à regarder est `available`, pas `free`.

Ce que contient chaque colonne de la ligne `Mem` :

| Colonne      | Contenu                                                                     |
|--------------|-----------------------------------------------------------------------------|
| `total`      | Mémoire utilisable au total                                                 |
| `used`       | Mémoire utilisée par les processus et le noyau, qui ne peut pas être libérée |
| `free`       | Mémoire qui ne sert à rien du tout                                          |
| `shared`     | Mémoire partagée, principalement les systèmes de fichiers `tmpfs`           |
| `buff/cache` | Buffers du noyau et cache de pages (contenu des fichiers lus ou écrits)      |
| `available`  | Estimation de la mémoire utilisable par une nouvelle application sans swapper |

Le calcul de `used` dépend de la version de procps-ng, le paquet qui fournit `free`. Depuis la version 4.0.1 (Debian 12, Ubuntu 24.04), `used` vaut `total - available`. Les versions plus anciennes (Ubuntu 22.04 par exemple) calculaient `total - free - buffers - cache`. Sur une machine récente, `used + free + buff/cache` ne tombe donc plus exactement sur `total`, c'est normal. Pour connaître votre version : `free --version`.

La ligne `Swap` est plus simple : taille totale du swap, partie utilisée et partie libre. Sur la machine de l'exemple, il n'y a pas de swap du tout, ce qui est fréquent sur les VM des hébergeurs cloud.

## Pourquoi la colonne free est presque toujours basse

Linux utilise la RAM inutilisée comme cache disque : quand un fichier est lu, son contenu reste en mémoire pour que la lecture suivante soit instantanée. C'est ce cache qui fait grimper `buff/cache` et baisser `free` au fil du temps. Dès qu'une application a besoin de mémoire, le noyau libère une partie de ce cache.

Une machine avec peu de `free` et beaucoup de `buff/cache` se porte donc très bien. Tout le cache n'est pas libérable pour autant (les fichiers stockés dans un `tmpfs`, comptés dans `shared`, ne peuvent pas être évincés) et le noyau garde une petite réserve : c'est pour cela que `available` est un peu inférieur à `free + buff/cache`.

Si quelqu'un vous dit que "le serveur n'a plus de RAM", regardez `available`. Tant qu'elle reste à plus de 10-15 % du total, il n'y a probablement pas de problème de mémoire.

Pour voir séparément les buffers et le cache, on utilise l'option `-w` (wide) :

```bash
free -wh
```

```
               total        used        free      shared     buffers       cache   available
Mem:            15Gi       5.7Gi       2.7Gi        13Mi        21Mi       7.7Gi        10Gi
Swap:             0B          0B          0B
```

Les `buffers` correspondent aux blocs bruts du disque gardés en mémoire par le noyau, en pratique surtout des métadonnées de systèmes de fichiers. Ils restent petits. Le `cache` regroupe le cache de pages (le contenu des fichiers) et la partie récupérable des caches internes du noyau, dont les caches d'inodes et d'entrées de répertoires. C'est lui qui peut atteindre plusieurs Go.

## Les options utiles

```bash
free -h              # Tailles lisibles (Ki, Mi, Gi)
free --si -h         # Puissances de 10 (M, G) au lieu de puissances de 2 (Mi, Gi)
free -m              # En mébioctets
free -g              # En gibioctets (arrondi, souvent trop grossier)
free -w              # Buffers et cache dans deux colonnes séparées
free -t              # Ajoute une ligne Total (RAM + swap)
free -h -s 2         # Rafraîchit l'affichage toutes les 2 secondes
free -h -s 10 -c 30  # 30 mesures espacées de 10 secondes, puis s'arrête
```

`-h` utilise des puissances de 2 (1 Gi = 1 073 741 824 octets) alors que `--si` utilise des puissances de 10 (1 G = 1 000 000 000 octets), soit environ 7 % d'écart sur les gigas : la même machine affiche `15Gi` avec `-h` et `16G` avec `--si -h`. Les barrettes de RAM étant vendues en puissances de 2, `-h` colle mieux à ce qui est installé dans la machine.

Pour suivre l'évolution de la mémoire en continu, `free -h -s 2` dépanne, mais `top`, `htop` ou `vmstat 1` donnent plus d'informations.

## Trouver les processus qui consomment la mémoire

Si `available` est bas, il faut trouver qui mange la mémoire. `ps` sait trier les processus par consommation :

```bash
ps -eo pid,user,%mem,rss,command --sort=-%mem | head -15
```

La colonne `RSS` (Resident Set Size) donne la mémoire physique occupée par chaque processus, en Kio. En interactif, `htop` avec un tri sur la colonne `MEM%` fait la même chose.

Pour être prévenu avant que la machine ne tombe à court de mémoire, un petit script suffit :

```bash
#!/usr/bin/env bash
# Affiche une alerte si la mémoire disponible passe sous un seuil (en %)

SEUIL_PCT=${1:-10}

read -r total available < <(free -m | awk '/^Mem:/ {print $2, $7}')
pct=$((available * 100 / total))

if [ "$pct" -lt "$SEUIL_PCT" ]; then
    echo "ALERTE : seulement ${available} Mo disponibles (${pct} % de ${total} Mo)"
fi
```

`$2` et `$7` sont les colonnes `total` et `available` de la ligne `Mem:`. Lancé toutes les minutes par cron (voir [Linux : Programmer une tâche avec cron]({% post_url 2025-10-11-Linux-programmer-une-tache-avec-cron %})), il n'écrit rien tant que tout va bien. Cron vous envoie la sortie par mail si un MTA est configuré, sinon on peut remplacer le `echo` par un appel à votre outil de notification.

## Le swap

Du swap utilisé n'est pas forcément mauvais signe. Le noyau déplace dans le swap les pages mémoire dont personne ne se sert, pour garder plus de RAM pour le cache et les applications actives. Ce qui pose problème, c'est quand le système passe son temps à écrire et relire le swap (on parle de *thrashing*) : là, tout ralentit.

Pour faire la différence, il faut regarder l'activité du swap avec `vmstat` :

```bash
vmstat 1 3
```

```
procs -----------memory---------- ---swap-- -----io---- -system-- -------cpu-------
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st gu
 0  0      0 2810440  22364 8062448    0    0   574 24871  410    1  3  6 90  1  0  0
 0  0      0 2810336  22364 8062452    0    0     0     0  203  208  0  0 100  0  0  0
 1  0      0 2810336  22364 8062452    0    0     0     0  225  315  0  0 100  0  0  0
```

Les colonnes `si` (swap in) et `so` (swap out) indiquent la quantité de mémoire lue et écrite dans le swap chaque seconde. Si elles restent régulièrement au-dessus de zéro, la machine manque de RAM. Ici elles sont à 0, tout va bien. Attention, la première ligne est une moyenne depuis le démarrage, ce sont les suivantes qui reflètent l'activité actuelle.

Si une machine n'a pas de swap, ou pas assez, on peut ajouter un fichier de swap sans toucher aux partitions. Sur ext4 :

```bash
# Créer un fichier swap de 4 Go
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Le rendre permanent
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

## Vider le cache, une fausse bonne idée

On trouve souvent cette commande sur les forums pour "libérer de la RAM" :

```bash
sudo sync && echo 3 | sudo tee /proc/sys/vm/drop_caches
```

La valeur écrite choisit ce qui est vidé : `1` pour le cache de pages, `2` pour les caches d'inodes et d'entrées de répertoires, `3` pour les deux.

Elle fait bien remonter la colonne `free`, mais elle ne rend aucune mémoire aux applications : le cache était déjà disponible pour elles. Par contre, les prochaines lectures de fichiers devront repasser par le disque. Cette commande est utile pour des benchmarks (mesurer des performances sans cache), pas en production.

## Aller plus loin avec /proc/meminfo

`free` ne fait que lire et mettre en forme le fichier `/proc/meminfo`, qui contient beaucoup plus de détails :

```bash
grep -E 'MemTotal|MemAvailable|Dirty|Slab|Committed' /proc/meminfo
```

```
MemTotal:       16480968 kB
MemAvailable:   15969924 kB
Dirty:               160 kB
Slab:              53612 kB
Committed_AS:     397028 kB
```

Quelques champs que `free` n'affiche pas :

| Champ             | Contenu                                                                         |
|-------------------|---------------------------------------------------------------------------------|
| `Dirty`           | Pages modifiées en mémoire, pas encore écrites sur le disque                    |
| `Slab`            | Caches internes du noyau                                                        |
| `SReclaimable`    | Partie du `Slab` que le noyau peut récupérer                                    |
| `Committed_AS`    | Mémoire promise aux processus, qui peut dépasser la RAM physique                |
| `SwapCached`      | Pages revenues du swap en RAM mais toujours présentes dans le swap              |
| `HugePages_Total` | Nombre de pages de grande taille (2 Mo sur x86-64), utilisées par certaines bases de données |

## Voir aussi

- [df/du : surveiller et analyser l'espace disque sous Linux]({% post_url 2026-03-01-Surveiller-espace-disque-avec-df-et-du %})
- [Linux : Programmer une tâche avec cron]({% post_url 2025-10-11-Linux-programmer-une-tache-avec-cron %})
- [Ripgrep (rg) : chercher rapidement dans le code]({% post_url 2026-02-16-Chercher-dans-le-code-rapidement-avec-ripgrep %})
- [man free](https://man7.org/linux/man-pages/man1/free.1.html)
- [man proc_meminfo](https://man7.org/linux/man-pages/man5/proc_meminfo.5.html)
- [Linux Ate My RAM](https://www.linuxatemyram.com/), l'explication de référence sur le cache mémoire
