---
layout: article
title: "df et du : surveiller et analyser l'espace disque sous Linux"
author: Pierre Chopinet
tags:
  - linux
  - df
  - du
  - cli
  - shell
  - bash
  - outils
  - sysadmin
  - disque
---

Un serveur qui tombe parce que le disque est plein à 100 %, c'est le genre de problème qu'on ne veut vivre qu'une seule fois. `df` et `du` sont les deux commandes de base pour comprendre où part l'espace disque sous Linux. `df` donne une vue d'ensemble des systèmes de fichiers montés, `du` permet de plonger dans les répertoires pour trouver ce qui prend de la place.
<!--more-->

Objectifs de l'article :
- Comprendre la différence entre `df` et `du`
- Maîtriser les options essentielles de chaque commande
- Savoir diagnostiquer rapidement un disque plein
- Découvrir des cas pratiques du quotidien (nettoyage, monitoring, scripts)
- Connaître les pièges courants et les alternatives modernes

---

## L'essentiel en 30 secondes

Vue d'ensemble des partitions :

```bash
df -h
```

Trouver les gros répertoires :

```bash
du -sh /var/* | sort -rh | head -20
```

Ces deux commandes suffisent pour diagnostiquer 90 % des problèmes d'espace disque. Le reste de cet article vous donne les clés pour aller plus loin.

---

## df : l'état des systèmes de fichiers

`df` (disk free) affiche l'espace total, utilisé et disponible de chaque système de fichiers monté. C'est la première commande à lancer quand on soupçonne un problème d'espace disque.

### Usage de base

```bash
df -h
```

Sortie typique :

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        50G   32G   16G  67% /
/dev/sda2       200G  180G   10G  95% /home
tmpfs           3.9G     0  3.9G   0% /dev/shm
/dev/sdb1       500G  420G   55G  89% /data
```

Le flag `-h` (human-readable) affiche les tailles en Ko, Mo, Go au lieu de blocs bruts. Sans cette option, les valeurs sont en blocs de 1 Ko, ce qui est peu lisible.

### Options courantes

```bash
df -h                     # Tailles lisibles (Go, Mo, Ko)
df -H                     # Tailles en puissances de 10 (1 Go = 1 000 000 000 octets)
df -T                     # Affiche le type de système de fichiers (ext4, xfs, tmpfs…)
df -i                     # Affiche l'utilisation des inodes au lieu de l'espace
df -h /home               # Uniquement le FS qui contient /home
df -h --total             # Ajoute une ligne de total en bas
df -x tmpfs -x devtmpfs   # Exclure les FS virtuels pour une vue plus propre
```

### Lire la sortie correctement

| Colonne     | Signification |
|-------------|---------------|
| `Filesystem` | Le device ou la partition |
| `Size`       | Taille totale du FS |
| `Used`       | Espace utilisé |
| `Avail`      | Espace disponible |
| `Use%`       | Pourcentage d'utilisation |
| `Mounted on` | Point de montage |

> Attention : `Used + Avail ≠ Size`. La différence vient de l'espace réservé pour root (typiquement 5 % sur ext4). Cet espace est invisible pour les utilisateurs normaux, mais `root` peut toujours écrire dessus quand le disque semble « plein ».

### Surveiller les inodes

Un disque peut afficher de l'espace libre mais être inutilisable si les inodes sont épuisés (beaucoup de petits fichiers) :

```bash
df -ih
```

```
Filesystem     Inodes IUsed IFree IUse% Mounted on
/dev/sda1        3.2M  320K  2.9M   10% /
/dev/sda2         13M   12M  800K   94% /home
```

Si `IUse%` approche 100 %, vous ne pourrez plus créer de fichiers même avec de l'espace disque libre. C'est un cas classique avec des applications qui génèrent des millions de petits fichiers temporaires (caches, sessions, etc.).

---

## du : analyser l'espace par répertoire

`du` (disk usage) calcule la taille d'un répertoire en parcourant récursivement son contenu. C'est l'outil pour répondre à la question « qu'est-ce qui prend autant de place ? ».

### Usage de base

```bash
du -sh /var
```

```
4.2G    /var
```

- `-s` : résumé (summary), affiche uniquement le total du répertoire
- `-h` : tailles lisibles

Sans `-s`, `du` affiche chaque sous-répertoire, ce qui peut être très verbeux sur des arborescences profondes.

### Explorer un répertoire

```bash
# Taille de chaque sous-répertoire immédiat
du -sh /var/*

# Pareil, mais trié du plus gros au plus petit
du -sh /var/* | sort -rh

# Limiter la profondeur de récursion
du -h --max-depth=1 /var

# Top 10 des plus gros répertoires
du -sh /var/* | sort -rh | head -10
```

Sortie typique :

```
2.1G    /var/log
1.5G    /var/cache
320M    /var/lib
180M    /var/backups
12M     /var/mail
4K      /var/tmp
```

On voit immédiatement que `/var/log` et `/var/cache` sont les principaux consommateurs. On peut ensuite descendre d'un niveau :

```bash
du -sh /var/log/* | sort -rh | head -10
```

### Options courantes

```bash
du -sh répertoire              # Résumé d'un répertoire
du -h --max-depth=1 /          # Vue d'ensemble du premier niveau
du -sh --apparent-size file    # Taille apparente (vs blocs alloués)
du -sh --exclude='*.log' /var  # Exclure un pattern
du -ch /home/user/*.tar.gz     # Total de fichiers spécifiques (-c ajoute un total)
du -a /var/log | sort -rn | head -20  # Top 20 des fichiers les plus gros (en blocs)
```

### Taille apparente vs blocs alloués

Par défaut, `du` affiche l'espace réellement occupé sur le disque (blocs alloués). Un fichier de 1 octet occupe quand même un bloc entier (4 Ko en général). Pour afficher la taille logique du contenu :

```bash
du --apparent-size -sh fichier
du -sh fichier
```

La différence est notable pour les fichiers sparse (bases de données, images disque VM), qui peuvent avoir une taille apparente de plusieurs Go mais n'occuper que quelques Mo réellement.

---

## df vs du : quand utiliser lequel ?

| Besoin | Commande | Pourquoi |
|--------|----------|----------|
| « Mon disque est-il plein ? » | `df -h` | Vue instantanée de tous les FS |
| « Les inodes sont-ils épuisés ? » | `df -ih` | Vérifie les inodes |
| « Qu'est-ce qui prend de la place ? » | `du -sh /répertoire/*` | Descend dans l'arborescence |
| « Combien pèse ce dossier ? » | `du -sh /chemin` | Calcul récursif |
| « Quel type de FS j'utilise ? » | `df -T` | Affiche ext4, xfs, etc. |

> `df` lit les métadonnées du système de fichiers : c'est instantané. `du` parcourt réellement l'arborescence : ça peut être lent sur des millions de fichiers.

---

## Pourquoi df et du peuvent donner des chiffres différents

C'est une source de confusion classique. `df` dit que le disque est plein à 95 %, mais la somme des `du` ne donne que 70 %. Plusieurs raisons possibles :

### Fichiers supprimés mais encore ouverts

Quand un processus garde un fichier ouvert après sa suppression (`rm`), le fichier disparaît du listing (`ls`, `du`) mais l'espace n'est pas libéré tant que le processus tourne. `df` voit l'espace occupé, `du` ne le voit plus.

Pour diagnostiquer :

```bash
# Trouver les fichiers supprimés encore ouverts
sudo lsof +L1
```

Solution : redémarrer le processus fautif, ou tronquer le fichier (si c'est un log) :

```bash
# Tronquer un log sans fermer le file descriptor
sudo truncate -s 0 /chemin/vers/le/fichier.log
```

### Espace réservé pour root

ext4 réserve par défaut 5 % de l'espace pour `root`. Cet espace n'est pas comptabilisé dans `Avail` de `df` pour les utilisateurs normaux, mais `du` ne le voit pas non plus.

Pour vérifier ou modifier ce réserve :

```bash
# Voir le pourcentage réservé
sudo tune2fs -l /dev/sda1 | grep 'Reserved block count'

# Réduire à 1 % (acceptable pour des partitions de données)
sudo tune2fs -m 1 /dev/sda1
```

### Snapshots et quotas

Les snapshots (LVM, ZFS, Btrfs) et les quotas de disque peuvent créer d'autres divergences entre ce que `df` et `du` rapportent.

---

## Cas pratiques

### Diagnostiquer un disque plein en urgence

La méthode classique en 3 commandes :

```bash
# Quelle partition est pleine ?
df -h

# Trouver les gros répertoires au premier niveau
du -sh /* 2>/dev/null | sort -rh | head -10

# Descendre dans le répertoire coupable
du -sh /var/* | sort -rh | head -10
```

On descend ainsi de niveau en niveau jusqu'à trouver le coupable (souvent un log qui a explosé, un cache non purgé, ou des backups oubliées).

### Trouver les fichiers les plus volumineux

```bash
# Top 20 des fichiers les plus gros sur tout le disque
sudo find / -type f -exec du -h {} + 2>/dev/null | sort -rh | head -20

# Plus rapide : utiliser du avec sort (évite le overhead de find + exec)
sudo du -ah / 2>/dev/null | sort -rh | head -20
```

### Surveiller l'espace disque dans un script

```bash
#!/usr/bin/env bash
# Alerte si un FS dépasse 90% d'utilisation

SEUIL=90

df -h --output=pcent,target | tail -n +2 | while read usage mount; do
    pct=${usage%\%}
    if [ "$pct" -gt "$SEUIL" ] 2>/dev/null; then
        echo "ALERTE : $mount est à ${usage} d'utilisation"
    fi
done
```

### Nettoyer les logs qui prennent trop de place

```bash
# Voir la taille des logs
du -sh /var/log/*  | sort -rh | head -10

# Nettoyer les journaux systemd de plus de 7 jours
sudo journalctl --vacuum-time=7d

# Limiter la taille totale des journaux à 500 Mo
sudo journalctl --vacuum-size=500M

# Trouver et supprimer les logs compressés anciens
sudo find /var/log -name '*.gz' -mtime +30 -delete
```

### Surveiller la taille d'un dossier de build/cache

```bash
# Taille du cache Docker
du -sh /var/lib/docker

# Taille des caches npm/pip/maven
du -sh ~/.npm ~/.cache/pip ~/.m2/repository 2>/dev/null

# Nettoyage Docker (images, containers et caches inutilisés)
docker system df
docker system prune -a
```

### Comparer l'espace avant/après une opération

```bash
# Avant nettoyage
df -h / | tail -1 | awk '{print "Avant:", $4, "disponibles"}'

# Votre nettoyage ici...
sudo apt autoremove -y && sudo apt clean

# Après nettoyage
df -h / | tail -1 | awk '{print "Après:", $4, "disponibles"}'
```

---

## Options essentielles

### df

| Option | Description |
|--------|-------------|
| `-h` | Tailles lisibles (Ko, Mo, Go) |
| `-H` | Tailles en puissances de 10 |
| `-T` | Affiche le type de FS |
| `-i` | Affiche les inodes |
| `-x TYPE` | Exclure un type de FS |
| `--total` | Ajoute une ligne de total |
| `--output=FIELDS` | Choisir les colonnes (source, fstype, size, used, avail, pcent, target) |

### du

| Option | Description |
|--------|-------------|
| `-s` | Résumé (total uniquement) |
| `-h` | Tailles lisibles |
| `--max-depth=N` | Limiter la profondeur |
| `-a` | Inclure les fichiers (pas seulement les répertoires) |
| `-c` | Ajouter un total |
| `--apparent-size` | Taille logique vs blocs alloués |
| `--exclude=PATTERN` | Exclure un pattern |
| `-x` | Rester sur le même système de fichiers |

L'option `-x` de `du` est particulièrement utile quand on scanne `/` et qu'on ne veut pas traverser les montages réseau ou les systèmes de fichiers virtuels (`/proc`, `/sys`, `/dev`).

---

## FAQ

**`df -h` montre le disque plein, mais `du -sh /` donne moins — pourquoi ?**
Des fichiers supprimés sont probablement encore ouverts par un processus. Utilisez `lsof +L1` pour les trouver. Voir la section « Pourquoi df et du peuvent donner des chiffres différents ».

**Comment libérer de l'espace rapidement en urgence ?**
Par ordre d'impact : nettoyez les logs (`journalctl --vacuum-size=500M`), purgez les caches (`apt clean`, `docker system prune`), supprimez les vieux backups. En dernier recours, trouvez les gros fichiers avec `du -ah / | sort -rh | head -20`.

**Peut-on utiliser `du` sur un répertoire distant (NFS, SSHFS) ?**
Oui, mais c'est lent car `du` doit traverser le réseau pour chaque fichier. Préférez lancer `du` directement sur la machine distante via SSH : `ssh serveur 'du -sh /data/*'`.

---

## Voir aussi

- [Ripgrep (rg) : chercher dans le code à la vitesse de l'éclair]({% post_url 2026-02-16-Chercher-dans-le-code-rapidement-avec-ripgrep %})
- [Sed : éditer des fichiers en ligne de commande avec des regex]({% post_url 2026-01-19-Sed-editer-des-fichiers-en-ligne-de-commande %})
- [Linux : Programmer une tâche avec cron]({% post_url 2025-10-11-Linux-programmer-une-tache-avec-cron %})
- [Comment manipuler du JSON en ligne de commande avec jq]({% post_url 2025-09-17-Comment-utiliser-jq %})

## Références

- [man df](https://man7.org/linux/man-pages/man1/df.1.html) et [man du](https://man7.org/linux/man-pages/man1/du.1.html)
- [ncdu - NCurses Disk Usage](https://dev.yorhel.nl/ncdu)
- [duf - Disk Usage/Free Utility](https://github.com/muesli/duf)
