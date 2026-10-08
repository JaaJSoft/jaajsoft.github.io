---
layout: article
title: "df/du : surveiller et analyser l'espace disque sous Linux"
description: "Surveiller l'espace disque sous Linux avec df et du : réserve root, inodes, recherche de ce qui prend la place et écarts entre les deux commandes."
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

Quand un disque se remplit, les symptômes sont rarement clairs : une base de données qui refuse d'écrire, des logs qui s'arrêtent, un `No space left on device` au milieu d'une compilation. `df` indique quel système de fichiers est plein, puis `du` permet de trouver ce qui prend la place. Voyons comment utiliser ces deux commandes, et pourquoi elles ne donnent pas toujours les mêmes chiffres.
<!--more-->

Dans cet article :
- df : l'état des systèmes de fichiers
- L'espace réservé à root
- Quand les inodes sont épuisés
- du : trouver ce qui prend de la place
- Pourquoi df et du ne donnent pas les mêmes chiffres
- Diagnostiquer un disque plein
- Faire de la place
- Surveiller l'espace disque avec un script
- ncdu et duf

## df : l'état des systèmes de fichiers

`df` (*disk free*) affiche l'espace total, utilisé et disponible de chaque système de fichiers monté. C'est la première commande à lancer quand on soupçonne un disque plein :

```bash
df -h
```

Sur un serveur dont le premier disque est partitionné en `/` et `/home`, avec un second disque monté sur `/data`, on obtient quelque chose comme :

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        50G   32G   16G  67% /
/dev/sda2       200G  180G   10G  95% /home
tmpfs           3.9G     0  3.9G   0% /dev/shm
/dev/sdb1       500G  420G   55G  89% /data
```

Chaque ligne donne le périphérique (`Filesystem`), sa taille (`Size`), l'espace utilisé (`Used`), l'espace encore disponible (`Avail`), le pourcentage d'utilisation (`Use%`) et le point de montage (`Mounted on`). Ici, c'est `/home` qu'il faut surveiller. `df` lit simplement les compteurs tenus par chaque système de fichiers : c'est pour cela qu'il répond instantanément.

Pour les exemples qui suivent, on utilise un système de fichiers ext4 de 100 Go créé dans un fichier image, ce qui permet de tout tester sans toucher à ses vrais disques (d'où le périphérique `/dev/loop0`) :

```bash
truncate -s 100G data.img
mkfs.ext4 -q data.img
sudo mkdir -p /mnt/data
sudo mount -o loop data.img /mnt/data
```

L'option `-h` affiche des tailles lisibles en puissances de 1024, alors que `-H` utilise des puissances de 1000. Sans option, `df` compte en blocs de 1 Kio, ce qui est peu lisible. Voici ce disque de test vu des trois façons :

```
$ df /mnt/data
Filesystem     1K-blocks  Used Available Use% Mounted on
/dev/loop0     102626232    24  97366944   1% /mnt/data
$ df -h /mnt/data
Filesystem      Size  Used Avail Use% Mounted on
/dev/loop0       98G   24K   93G   1% /mnt/data
$ df -H /mnt/data
Filesystem      Size  Used Avail Use% Mounted on
/dev/loop0      106G   25k  100G   1% /mnt/data
```

Les autres options utiles :

```bash
df -T                            # Ajoute le type de système de fichiers (ext4, xfs, tmpfs...)
df -i                            # Utilisation des inodes au lieu de l'espace
df -h /home                      # Seulement le système de fichiers qui contient /home
df -h --total                    # Ajoute une ligne de total
df -h -x tmpfs -x devtmpfs       # Sans les systèmes de fichiers en mémoire
df --output=source,pcent,target  # Choisir les colonnes affichées
```

Sur Ubuntu, `df` masque déjà par défaut les systèmes de fichiers `squashfs` (les snaps) et `devtmpfs`, grâce à un patch propre à la distribution. L'option `-a` les affiche quand même.

## L'espace réservé à root

Revenons sur la sortie de `df -h` pour le disque de test :

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/loop0       98G   24K   93G   1% /mnt/data
```

Le système de fichiers est vide, et pourtant il manque 5 Go entre `Size` et `Avail`. Ce sont les 5 % qu'ext4 réserve par défaut au super-utilisateur. D'après la page de manuel de `mke2fs`, cette réserve limite la fragmentation et permet aux services qui tournent en root, comme `syslogd`, de continuer à fonctionner quand les utilisateurs normaux ne peuvent plus écrire. `Used + Avail` ne fait donc pas tout à fait `Size`, et un disque à 100 % a encore un peu de place pour root.

D'ailleurs, `Use%` n'est pas calculé sur `Size` mais sur l'espace accessible aux utilisateurs normaux : c'est `Used / (Used + Avail)`, arrondi à l'unité supérieure.

`tune2fs` permet de voir la taille de la réserve, en blocs :

```bash
sudo tune2fs -l /dev/loop0 | grep -iE 'block (count|size)'
```

```
Block count:              26214400
Reserved block count:     1310720
Block size:               4096
```

Soit 1 310 720 blocs de 4 Kio, c'est-à-dire 5 Gio. Sur une partition de données, qui ne contient pas le système, on peut réduire cette réserve avec l'option `-m` :

```bash
sudo tune2fs -m 1 /dev/loop0
```

```
tune2fs 1.47.0 (5-Feb-2023)
Setting reserved blocks percentage to 1% (262144 blocks)
```

Et l'espace disponible passe de 93 à 97 Go :

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/loop0       98G   24K   97G   1% /mnt/data
```

Pour la suite, on remet la valeur par défaut avec `sudo tune2fs -m 5 /dev/loop0`.

## Quand les inodes sont épuisés

Chaque fichier utilise un inode, la structure qui stocke ses métadonnées (propriétaire, droits, dates, emplacement des données). Sur ext4, le nombre d'inodes est fixé à la création du système de fichiers et n'augmente que si on l'agrandit. Avec des millions de petits fichiers (cache, sessions, fichiers temporaires...), il peut donc devenir impossible de créer un fichier alors qu'il reste de la place.

Pour le montrer, voici un système de fichiers de 1 Go créé avec seulement 20 000 inodes (`mkfs.ext4 -N 20000`), puis rempli de fichiers vides :

```
$ touch /mnt/cache/sessions/nouveau
touch: cannot touch '/mnt/cache/sessions/nouveau': No space left on device
$ df -h /mnt/cache
Filesystem      Size  Used Avail Use% Mounted on
/dev/loop1      985M  544K  917M   1% /mnt/cache
```

L'erreur parle d'espace, mais `df -h` indique encore 917 Mo disponibles. C'est `df -i` (ou `df -ih` pour des nombres lisibles) qui donne l'explication :

```
$ df -ih /mnt/cache
Filesystem     Inodes IUsed IFree IUse% Mounted on
/dev/loop1        20K   20K     0  100% /mnt/cache
```

Pour trouver le répertoire qui contient tous ces fichiers, `du` sait compter les inodes au lieu des octets :

```bash
sudo du --inodes -x /mnt/cache | sort -rn | head -5
```

```
20087	/mnt/cache
20085	/mnt/cache/sessions
1	/mnt/cache/lost+found
```

## du : trouver ce qui prend de la place

`du` (*disk usage*) parcourt un répertoire et additionne la taille de tout ce qu'il contient :

```bash
sudo du -sh /var
```

```
127M	/var
```

L'option `-s` n'affiche que le total (sans elle, `du` liste chaque sous-répertoire, ce qui devient vite très long) et `-h` donne des tailles lisibles. On utilise `sudo` pour pouvoir lire tous les répertoires.

Pour voir la taille de chaque sous-répertoire, du plus gros au plus petit, on trie la sortie avec `sort -rh`, qui sait comparer des tailles comme `4.0K`, `2.5M` ou `123M` :

```bash
sudo du -sh /var/* | sort -rh | head -5
```

```
123M	/var/lib
2.5M	/var/cache
1.2M	/var/log
28K	/var/spool
4.0K	/var/tmp
```

C'est `/var/lib` qui prend presque toute la place, on descend donc d'un niveau :

```bash
sudo du -sh /var/lib/* | sort -rh | head -5
```

```
52M	/var/lib/apt
39M	/var/lib/postgresql
29M	/var/lib/dpkg
4.0M	/var/lib/ucf
376K	/var/lib/systemd
```

Les autres options utiles :

```bash
du -h -d 1 /var                  # Un seul niveau de sous-répertoires (comme --max-depth=1)
du -sh --exclude='*.log' /var    # Exclure les fichiers qui correspondent à un motif
du -ch *.tar.gz                  # Taille de plusieurs fichiers, avec une ligne de total
du -shx /                        # Rester sur le même système de fichiers
du -sh --apparent-size fichier   # Taille du contenu plutôt qu'espace occupé
du --inodes -d 1 /var            # Nombre d'inodes au lieu de la taille
```

Attention, contrairement à `df`, `du` parcourt réellement toute l'arborescence : sur des millions de fichiers, cela peut prendre du temps. Sur un partage réseau (NFS, SSHFS), il doit en plus récupérer les informations de chaque fichier à travers le réseau. Il vaut mieux alors le lancer directement sur la machine distante, par exemple avec `ssh serveur 'du -sh /data/*'`.

Par défaut, `du` affiche l'espace réellement occupé sur le disque, qui se compte en blocs. Un fichier de 2 octets occupe donc un bloc entier, 4 Kio sur la plupart des systèmes ext4. L'option `--apparent-size` donne la taille du contenu :

```
$ du -h petit.txt
4.0K	petit.txt
$ du -h --apparent-size petit.txt
2	petit.txt
```

À l'inverse, un fichier creux (*sparse*), comme une image disque de machine virtuelle, peut annoncer une taille énorme sans occuper autant de place, car ses zones vides ne sont pas stockées :

```
$ truncate -s 10G creux.img
$ du -h creux.img
0	creux.img
$ du -h --apparent-size creux.img
10G	creux.img
```

`ls -lh` affiche lui aussi 10G, c'est-à-dire la taille apparente. L'image de 100 Go utilisée plus haut n'occupait d'ailleurs que 518 Mo sur le disque juste après sa création.

## Pourquoi df et du ne donnent pas les mêmes chiffres

C'est une source de confusion classique : `df` annonce un disque presque plein, mais la somme des `du` est loin du compte.

### Des fichiers supprimés mais encore ouverts

Un fichier supprimé alors qu'un processus le garde ouvert disparaît de l'arborescence, mais son contenu reste sur le disque jusqu'à ce que le processus le ferme. `du` ne le voit plus, `df` le compte toujours. Cela arrive souvent avec un fichier de log supprimé pendant que l'application tourne.

Pour le reproduire, on crée un log de 1 Go sur le disque de test, on le suit avec `tail -f` dans un autre terminal, puis on le supprime avec `rm` :

```
$ df -h /mnt/data
Filesystem      Size  Used Avail Use% Mounted on
/dev/loop0       98G  1.1G   92G   2% /mnt/data
$ sudo du -sh /mnt/data
20K	/mnt/data
```

`lsof +L1` (paquet `lsof`) liste les fichiers ouverts qui n'ont plus aucun nom sur le disque. Par défaut, `lsof` affiche les fichiers qui correspondent à l'un ou l'autre de ses critères. L'option `-a` demande de les combiner, ce qui permet de se limiter au système de fichiers qui nous intéresse :

```bash
sudo lsof -a +L1 /mnt/data
```

```
COMMAND  PID USER   FD   TYPE DEVICE   SIZE/OFF NLINK NODE NAME
tail    4867 root    3r   REG    7,0 1073741824     0   12 /mnt/data/app.log (deleted)
```

Le plus propre est de redémarrer le processus fautif, qui fermera le fichier. Si ce n'est pas possible, on peut vider le fichier en passant par le descripteur que le processus a ouvert, dans `/proc/PID/fd/` (ici le PID 4867 et le descripteur 3, colonne `FD`) :

```bash
sudo truncate -s 0 /proc/4867/fd/3
```

```
$ df -h /mnt/data
Filesystem      Size  Used Avail Use% Mounted on
/dev/loop0       98G   24K   93G   1% /mnt/data
```

Pour éviter le problème, mieux vaut vider un log en cours d'utilisation que le supprimer : `sudo truncate -s 0 /var/log/mon-app.log` libère la place, même si l'application le garde ouvert.

### Des fichiers cachés sous un point de montage

Si des fichiers ont été écrits dans un répertoire avant qu'un autre disque soit monté dessus, ils deviennent invisibles mais occupent toujours de la place. Cela arrive par exemple avec un script de sauvegarde lancé alors que le disque de destination n'était pas monté. Ici, 500 Mo ont été écrits dans `/mnt/data/backups` avant qu'un autre disque y soit monté :

```
$ df -h /mnt/data
Filesystem      Size  Used Avail Use% Mounted on
/dev/loop0       98G  501M   93G   1% /mnt/data
$ sudo du -shx /mnt/data
20K	/mnt/data
```

Pour voir ce qui se cache dessous, on monte le système de fichiers une deuxième fois ailleurs avec `mount --bind`. Ce deuxième montage ne contient pas les disques montés dans ses sous-répertoires, si bien que le contenu caché y redevient visible :

```bash
sudo mkdir /mnt/vue
sudo mount --bind /mnt/data /mnt/vue
sudo du -sh /mnt/vue/backups
```

```
501M	/mnt/vue/backups
```

La même technique fonctionne pour la racine, avec `sudo mount --bind / /mnt/vue`. Une fois le ménage fait, `sudo umount /mnt/vue` retire le second montage.

### Des snapshots

Sur Btrfs ou ZFS, un snapshot partage ses données avec le système de fichiers d'origine. Un fichier supprimé qui existe encore dans un snapshot continue donc d'occuper de la place, jusqu'à la suppression du dernier snapshot qui le contient.

## Diagnostiquer un disque plein

Une fois le système de fichiers plein repéré avec `df -h`, on cherche les plus gros répertoires en restant sur ce système de fichiers grâce à l'option `-x` :

```bash
sudo du -xh -d 1 / | sort -rh | head -10
```

Sans `-x`, `du` parcourrait aussi `/proc`, `/sys` et les autres disques montés. On descend ensuite de niveau en niveau, avec par exemple `sudo du -xh -d 1 /var | sort -rh | head -10`, jusqu'à trouver le coupable : souvent des logs, un cache ou de vieilles sauvegardes.

Pour lister directement les plus gros fichiers, `find` est plus adapté :

```bash
sudo find / -xdev -type f -size +100M -exec du -h {} + 2>/dev/null | sort -rh | head -20
```

`-xdev` joue le même rôle que `-x` pour `du`, et `-size +100M` ne garde que les fichiers de plus de 100 Mio. On pourrait être tenté par `du -ah / | sort -rh | head -20`, mais la liste serait remplie de répertoires : la taille d'un répertoire comprend celle de son contenu, il passe donc toujours devant ses propres fichiers.

## Faire de la place

Les logs sont souvent les premiers suspects. Pour les journaux de systemd :

```bash
# Taille occupée par les journaux
sudo journalctl --disk-usage

# Supprimer les journaux de plus de 7 jours
sudo journalctl --vacuum-time=7d

# Ou réduire leur taille totale à 500 Mo
sudo journalctl --vacuum-size=500M
```

Et pour les anciens logs compressés par logrotate :

```bash
sudo find /var/log -name '*.gz' -mtime +30 -delete
```

Côté paquets, `apt autoremove` supprime les dépendances qui ne servent plus et `apt clean` vide le cache des paquets téléchargés (`/var/cache/apt/archives`) :

```bash
sudo apt autoremove
sudo apt clean
```

Sur une machine qui fait tourner Docker, les images et les conteneurs s'accumulent dans `/var/lib/docker` :

```bash
# Taille totale
sudo du -sh /var/lib/docker

# Détail par images, conteneurs, volumes et cache de build
docker system df

# Supprimer les conteneurs arrêtés, les réseaux et les images inutilisés, et le cache de build
docker system prune -a
```

Attention, avec `-a`, `docker system prune` supprime toutes les images qui ne sont utilisées par aucun conteneur, pas seulement les images orphelines : il faudra les retélécharger. Les volumes ne sont pas touchés, sauf avec l'option `--volumes`.

Sur un poste de développement, les caches des gestionnaires de paquets peuvent aussi peser lourd :

```bash
du -sh ~/.npm ~/.cache/pip ~/.m2/repository 2>/dev/null
```

## Surveiller l'espace disque avec un script

Pour être prévenu avant que le disque soit plein, un petit script suffit :

```bash
#!/usr/bin/env bash
# Affiche une alerte pour chaque système de fichiers qui dépasse un seuil d'utilisation (en %)

SEUIL=${1:-90}

df --output=pcent,target | tail -n +2 | while read -r usage mount; do
    pct=${usage%\%}
    if [ "$pct" -gt "$SEUIL" ] 2>/dev/null; then
        echo "ALERTE : $mount est à ${usage} d'utilisation"
    fi
done
```

`--output=pcent,target` ne garde que le pourcentage d'utilisation et le point de montage, `tail -n +2` retire la ligne d'en-tête et `${usage%\%}` enlève le `%` pour pouvoir comparer les nombres. Le seuil vaut 90 % par défaut, ou la valeur passée en argument.

Le script n'écrit rien tant que tout va bien. On peut le lancer régulièrement avec cron (voir [Linux : Programmer une tâche avec cron]({% post_url 2025-10-11-Linux-programmer-une-tache-avec-cron %})), qui vous enverra sa sortie par mail si un MTA est configuré.

## ncdu et duf

`df` et `du` sont disponibles partout, mais deux outils rendent l'exploration plus confortable.

`ncdu` (*NCurses Disk Usage*) scanne un répertoire puis permet de parcourir l'arborescence triée par taille, et de supprimer un fichier ou un répertoire avec la touche `d`. On l'installe avec `sudo apt install ncdu` et on le lance avec `sudo ncdu -x /`.

`duf` (*Disk Usage/Free*) remplace `df` avec un affichage en couleurs, regroupé en tableaux par type de périphérique (disques locaux, réseau, FUSE, systèmes de fichiers spéciaux...). Il est dans les dépôts depuis Ubuntu 22.04 et Debian 12 : `sudo apt install duf`.

## Voir aussi

- [free : surveiller et comprendre l'utilisation mémoire sous Linux]({% post_url 2026-03-16-Surveiller-la-memoire-avec-free %})
- [Linux : Programmer une tâche avec cron]({% post_url 2025-10-11-Linux-programmer-une-tache-avec-cron %})
- [man df](https://man7.org/linux/man-pages/man1/df.1.html) et [man du](https://man7.org/linux/man-pages/man1/du.1.html)
- [ncdu](https://dev.yorhel.nl/ncdu) et [duf](https://github.com/muesli/duf)
