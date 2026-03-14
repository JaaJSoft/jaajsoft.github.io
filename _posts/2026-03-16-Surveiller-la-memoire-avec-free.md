---
layout: article
title: "free : surveiller et comprendre l'utilisation mémoire sous Linux"
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

Un processus qui se fait tuer par l'OOM killer, une application qui rame sans raison apparente, un serveur qui swap en permanence : la plupart de ces problèmes commencent par un manque de visibilité sur l'utilisation mémoire. `free` est la commande la plus rapide pour obtenir un instantané de la mémoire vive et du swap sous Linux.
<!--more-->

Objectifs de l'article :
- Comprendre la sortie de `free` et la signification de chaque colonne
- Distinguer mémoire utilisée, buffers, cache et mémoire réellement disponible
- Savoir diagnostiquer rapidement un problème de mémoire
- Découvrir des cas pratiques du quotidien (monitoring, alertes, scripts)
- Comprendre les pièges courants liés au cache et au swap

---

## L'essentiel en 30 secondes

État de la mémoire en un coup d'oeil :

```bash
free -h
```

Sortie typique :

```
               total        used        free      shared  buff/cache   available
Mem:            15Gi       6.2Gi       1.3Gi       512Mi       8.1Gi       8.8Gi
Swap:          4.0Gi       0.0Gi       4.0Gi
```

Le chiffre qui compte vraiment, c'est **`available`** (ici 8.8 Gi), pas `free`. Si `available` est proche de zéro et que le swap est fortement utilisé, votre machine manque de mémoire. Le reste de cet article vous explique pourquoi.

---

## Comprendre la sortie de free

### Les colonnes de la ligne Mem

| Colonne      | Signification                                                                                     |
|--------------|---------------------------------------------------------------------------------------------------|
| `total`      | Mémoire physique totale installée                                                                 |
| `used`       | Mémoire utilisée (total - free - buffers - cache)                                                 |
| `free`       | Mémoire totalement inutilisée                                                                     |
| `shared`     | Mémoire partagée (principalement tmpfs)                                                           |
| `buff/cache` | Mémoire utilisée par les buffers du noyau et le cache de pages                                    |
| `available`  | Estimation de la mémoire disponible pour démarrer de nouvelles applications, sans toucher au swap |

> **Attention** : ne confondez pas `free` et `available`. Une machine avec 1 Go de `free` mais 8 Go de `buff/cache` a en réalité ~9 Go de mémoire disponible, car le noyau peut libérer le cache à tout moment si une application en a besoin.

### La ligne Swap

| Colonne | Signification                   |
|---------|---------------------------------|
| `total` | Taille totale du swap configuré |
| `used`  | Swap actuellement utilisé       |
| `free`  | Swap encore disponible          |

Du swap utilisé n'est pas forcément un problème. Le noyau déplace en swap les pages mémoire rarement accédées pour libérer de la RAM pour le cache et les applications actives. C'est un problème uniquement quand le système swap activement (thrashing), ce qui se traduit par un ralentissement notable.

---

## free vs le vrai état de la mémoire

### Pourquoi « free » est faible même quand tout va bien

Linux utilise la RAM libre comme cache de pages pour accélérer les lectures/écritures disque. C'est un comportement normal et souhaitable. Une machine avec peu de mémoire `free` mais beaucoup de `buff/cache` fonctionne parfaitement : le noyau libérera le cache dès qu'une application aura besoin de mémoire.

```bash
# Ces deux commandes montrent la différence
free -h          # Mémoire "free" souvent faible
cat /proc/meminfo | grep -i available  # Mémoire réellement disponible
```

> **Conseil** : si quelqu'un dit « le serveur n'a plus de RAM », vérifiez la colonne `available`. Si elle est au-dessus de 10-15 % du total, il n'y a probablement pas de problème mémoire.

### Buffers vs cache

- **Buffers** : cache des métadonnées du système de fichiers (table d'inodes, entrées de répertoires, superblocs). En général quelques centaines de Mo.
- **Cache** : cache du contenu des fichiers (pages lues depuis le disque). Peut atteindre plusieurs Go sur une machine avec beaucoup de RAM.

Les deux sont libérables par le noyau quand la mémoire est nécessaire. C'est pour ça que `available ≈ free + buff/cache` (en simplifiant, car certains caches ne sont pas libérables).

---

## Options de free

```bash
free -h              # Tailles lisibles (Gi, Mi, Ki)
free -h -s 5         # Rafraîchir toutes les 5 secondes
free -h -c 10        # Afficher 10 fois puis quitter (combiner avec -s)
free -h -s 2 -c 5    # 5 mesures toutes les 2 secondes
free -m              # Afficher en mébioctets (Mio)
free -g              # Afficher en gibioctets (Gio)
free -b              # Afficher en octets
free --si            # Utiliser les puissances de 10 (Go, Mo) au lieu de 2 (Gio, Mio)
free -w              # Séparer buffers et cache en deux colonnes distinctes
free -t              # Ajouter une ligne Total (Mem + Swap)
free -l              # Afficher les statistiques low/high memory (surtout utile sur systèmes 32 bits)
```

L'option `-w` (wide) est particulièrement utile pour distinguer les buffers du cache quand on diagnostique un problème :

```bash
free -wh
```

```
               total        used        free      shared     buffers       cache   available
Mem:            15Gi       6.2Gi       1.3Gi       512Mi       320Mi       7.8Gi       8.8Gi
Swap:          4.0Gi          0B       4.0Gi
```

---

## Cas pratiques

### Diagnostiquer un serveur qui rame

```bash
# Vue d'ensemble rapide
free -h

# Si available est faible, identifier les processus gourmands
ps aux --sort=-%mem | head -15

# Version plus lisible avec les colonnes essentielles
ps -eo pid,user,%mem,%cpu,rss,command --sort=-%mem | head -15
```

La colonne `RSS` (Resident Set Size) de `ps` indique la mémoire physique réellement occupée par chaque processus en Ko.

### Surveiller la mémoire en continu

```bash
# Rafraîchissement toutes les 2 secondes, indéfiniment
free -h -s 2

# 30 mesures toutes les 10 secondes (5 minutes de monitoring)
free -h -s 10 -c 30
```

> Pour un monitoring plus riche en temps réel, `top`, `htop` ou `vmstat 1` offrent davantage de détails.

### Script d'alerte mémoire

```bash
#!/usr/bin/env bash
# Alerte si la mémoire disponible descend sous un seuil

SEUIL_PCT=10

total=$(free -m | awk '/^Mem:/ {print $2}')
available=$(free -m | awk '/^Mem:/ {print $7}')
pct=$((available * 100 / total))

if [ "$pct" -lt "$SEUIL_PCT" ]; then
    echo "ALERTE : seulement ${available} Mo disponibles (${pct}% de ${total} Mo)"
fi
```

Ce script peut être lancé via cron toutes les minutes pour une surveillance simple.

### Vider le cache manuellement

```bash
# Vider le cache de pages (page cache)
sudo sync && echo 3 | sudo tee /proc/sys/vm/drop_caches

# Valeurs possibles :
# 1 = page cache uniquement
# 2 = dentries et inodes
# 3 = page cache + dentries + inodes
```

> **Attention** : vider le cache ne libère pas réellement de la mémoire pour les applications — le cache **est** de la mémoire disponible. Cette opération est surtout utile pour des benchmarks (mesurer les performances sans cache) ou pour diagnostiquer un problème de cache spécifique. En production, laissez le noyau gérer le cache.

### Vérifier si le système utilise le swap activement

```bash
# Voir le swap utilisé
free -h

# Détail : activité swap en temps réel
vmstat 1 5
```

Dans la sortie de `vmstat`, les colonnes `si` (swap in) et `so` (swap out) indiquent le nombre de pages échangées par seconde. Si ces valeurs sont régulièrement supérieures à zéro, le système swap activement - c'est à ce moment que les performances se dégradent.

```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  0  10240  13200  32040 812000    0    0    12     8  150  300  5  2 93  0  0
 1  0  10240  12800  32040 812400    0    0     0     4  140  280  3  1 96  0  0
```

Ici `si=0` et `so=0` : pas de swap actif, tout va bien.

### Comparer la mémoire avant/après le lancement d'une application

```bash
# Avant
free -m | awk '/^Mem:/ {print "Avant : " $7 " Mo disponibles"}'

# Lancer l'application
./mon-application &

# Attendre que l'application se stabilise
sleep 10

# Après
free -m | awk '/^Mem:/ {print "Après : " $7 " Mo disponibles"}'
```

---

## /proc/meminfo : aller plus loin que free

`free` lit ses données depuis `/proc/meminfo`. Ce fichier contient bien plus de détails :

```bash
cat /proc/meminfo
```

Quelques champs utiles que `free` n'affiche pas directement :

| Champ             | Signification                                                                               |
|-------------------|---------------------------------------------------------------------------------------------|
| `MemAvailable`    | Estimation de la mémoire disponible (source de la colonne `available` de `free`)            |
| `Dirty`           | Pages modifiées en cache, pas encore écrites sur disque                                     |
| `Slab`            | Mémoire utilisée par les caches internes du noyau                                           |
| `SReclaimable`    | Partie du Slab qui peut être récupérée                                                      |
| `Committed_AS`    | Mémoire totale engagée (promise aux applications, potentiellement plus que la RAM physique) |
| `SwapCached`      | Swap lu en cache dans la RAM (pour des accès plus rapides si besoin)                        |
| `HugePages_Total` | Nombre de pages de taille supérieure (Huge Pages, utilisées par les bases de données)       |

```bash
# Voir uniquement les champs qui vous intéressent
grep -E 'MemTotal|MemAvailable|Dirty|Slab|Committed' /proc/meminfo
```

---

## FAQ

**La colonne `free` est presque à zéro, est-ce grave ?**
Non, c'est normal. Linux utilise la RAM libre comme cache disque. Regardez la colonne `available` : c'est elle qui indique la mémoire réellement disponible pour de nouvelles applications.

**Mon serveur utilise du swap, dois-je ajouter de la RAM ?**
Pas nécessairement. Du swap utilisé est normal : le noyau y déplace les pages rarement utilisées. C'est un problème uniquement si les colonnes `si`/`so` de `vmstat` montrent un swap actif constant (thrashing), ce qui se traduit par des lenteurs.

**Comment savoir quel processus consomme le plus de mémoire ?**
`ps aux --sort=-%mem | head -15` trie les processus par utilisation mémoire décroissante. Pour un suivi en temps réel, utilisez `htop` avec un tri par colonne MEM%.

**Quelle est la différence entre `free -h` et `free --si` ?**
`-h` utilise les puissances de 2 (1 Gi = 1 073 741 824 octets), `--si` utilise les puissances de 10 (1 G = 1 000 000 000 octets). La différence est d'environ 7 % sur les gibioctets. Les fabricants de RAM utilisent les puissances de 2, donc `-h` correspond mieux à la réalité matérielle.

**Comment augmenter le swap ?**
Vous pouvez ajouter un fichier de swap sans repartitionner :

```bash
# Créer un fichier swap de 4 Go
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Rendre permanent (ajouter dans /etc/fstab)
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

---

## Voir aussi

- [df/du : surveiller et analyser l'espace disque sous Linux]({% post_url 2026-03-01-Surveiller-espace-disque-avec-df-et-du %})
- [Ripgrep (rg) : chercher dans le code à la vitesse de l'éclair]({% post_url 2026-02-16-Chercher-dans-le-code-rapidement-avec-ripgrep %})
- [Linux : Programmer une tâche avec cron]({% post_url 2025-10-11-Linux-programmer-une-tache-avec-cron %})

## Références

- [man free](https://man7.org/linux/man-pages/man1/free.1.html)
- [man proc - /proc/meminfo](https://man7.org/linux/man-pages/man5/proc_meminfo.5.html)
- [Linux Ate My RAM](https://www.linuxatemyram.com/) — explication pédagogique du cache mémoire
- [vmstat - man page](https://man7.org/linux/man-pages/man8/vmstat.8.html)
