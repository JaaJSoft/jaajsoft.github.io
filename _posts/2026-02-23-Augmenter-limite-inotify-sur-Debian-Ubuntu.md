---
layout: article
title: "Augmenter la limite inotify sur Debian et Ubuntu"
author: Pierre Chopinet
tags:
  - linux
  - debian
  - ubuntu
  - inotify
---

Lorsque vous développez sur Linux avec des outils modernes (IDE, hot-reload, watchers), vous rencontrez souvent l'erreur « too many open files » ou « inotify watch limit reached ». Ces messages indiquent que la limite par défaut d'inotify est atteinte. Ce guide vous montre comment augmenter ces limites sur Debian et Ubuntu.
<!--more-->

Dans cet article, vous découvrirez :
- Ce qu'est inotify et pourquoi les limites existent
- Comment vérifier vos limites actuelles
- Comment augmenter temporairement les limites
- Comment rendre les modifications permanentes
- Les valeurs recommandées selon votre usage
- Comment diagnostiquer les processus qui consomment des watchers

---

## 1) Qu'est-ce qu'inotify ?

`inotify` (inode notify) est un sous-système du noyau Linux qui surveille les modifications du système de fichiers. Il permet aux applications de recevoir des notifications lorsqu'un fichier ou répertoire est créé, modifié, supprimé ou déplacé.


## 2) Les trois limites d'inotify

Le système inotify possède trois paramètres configurables :

### max_user_watches

**Nombre maximum de fichiers et répertoires surveillés par utilisateur**.

```bash
cat /proc/sys/fs/inotify/max_user_watches
# Par défaut : 8192 ou 524288 (selon distribution)
```

**Problème courant** : dépassement lors du watch de gros projets (node_modules, monorepos).

### max_user_instances

**Nombre maximum d'instances inotify par utilisateur** (nombre de processus pouvant surveiller des fichiers).

```bash
cat /proc/sys/fs/inotify/max_user_instances
# Par défaut : 128
```

**Problème courant** : trop d'outils tournent simultanément (IDE + webpack + tests + docker).

### max_queued_events

**Nombre maximum d'événements en attente dans la file** avant qu'ils ne soient traités.

```bash
cat /proc/sys/fs/inotify/max_queued_events
# Par défaut : 16384
```

**Problème courant** : modifications massives (git checkout, npm install) peuvent saturer la file.

---

## 3) Vérifier vos limites actuelles

### Afficher toutes les limites

```bash
sysctl fs.inotify
```

**Sortie typique** :
```
fs.inotify.max_queued_events = 16384
fs.inotify.max_user_instances = 128
fs.inotify.max_user_watches = 524288
```

### Vérifier combien de watchers vous utilisez

```bash
find /proc/*/fd -lname 'anon_inode:inotify' 2>/dev/null | wc -l
```

Affiche le nombre total d'instances inotify actives sur le système.

### Identifier les processus consommateurs

```bash
for pid in $(ps -ef | awk '{print $2}'); do
    count=$(find /proc/$pid/fd -lname 'anon_inode:inotify' 2>/dev/null | wc -l)
    if [ $count -gt 0 ]; then
        echo "$count watchers: $(ps -p $pid -o comm=)"
    fi
done | sort -rn
```

Cette commande liste les processus avec leurs nombres de watchers (utile pour diagnostiquer).

---

## 4) Augmenter les limites temporairement

### Modifier pour la session en cours

```bash
sudo sysctl fs.inotify.max_user_watches=524288
sudo sysctl fs.inotify.max_user_instances=512
sudo sysctl fs.inotify.max_queued_events=32768
```

**⚠️ Limitation** : ces changements sont perdus au redémarrage.

**Usage** : test rapide ou débogage ponctuel.

---

## 5) Augmenter les limites de façon permanente

### Méthode recommandée : créer un fichier de configuration

```bash
sudo nano /etc/sysctl.d/60-inotify.conf
```

Ajoutez ces lignes :

```
fs.inotify.max_user_watches=524288
fs.inotify.max_user_instances=512
fs.inotify.max_queued_events=32768
```

### Appliquer immédiatement sans redémarrage

```bash
sudo sysctl -p /etc/sysctl.d/60-inotify.conf
```

### Vérifier l'application

```bash
sysctl fs.inotify.max_user_watches
```

**Résultat attendu** :
```
fs.inotify.max_user_watches = 524288
```

---

## 6) Impact mémoire des limites

Chaque watcher consomme environ **1 Ko de RAM** (noyau Linux).

**Calcul pour 524 288 watchers** :
```
524288 watchers × 1 Ko = 512 Mo RAM
```

Sur une machine de développement moderne (16+ Go RAM), c'est négligeable.

**⚡ Conseil** : ne dépassez pas 4 millions de watchers sauf cas extrême (risque de ralentissements).

---

## 7) Debugging : lister les watchers actifs

### Méthode 1 : avec inotify-tools

Installez le paquet :
```bash
sudo apt update
sudo apt install inotify-tools
```

Surveillez en temps réel :
```bash
inotifywatch -v /chemin/vers/dossier
```

### Méthode 2 : script

Créez `list-watchers.sh` :
```bash
#!/bin/bash
for pid in /proc/*/fd/*; do
    if [ -L "$pid" ]; then
        link=$(readlink "$pid" 2>/dev/null)
        if [ "$link" = "anon_inode:inotify" ]; then
            echo "$(ps -p $(echo $pid | cut -d'/' -f3) -o comm=)"
        fi
    fi
done | sort | uniq -c | sort -rn
```

Exécutez :
```bash
chmod +x list-watchers.sh
./list-watchers.sh
```

**Sortie exemple** :
```
  45 node
  12 java
   8 code
   3 docker
```

---

## 8) Bonnes pratiques

### ✅ À faire

1. **Augmenter progressivement** : commencez par 524288, augmentez si nécessaire
2. **Monitorer la RAM** : surveillez l'impact mémoire avec `htop` ou `free -h`
3. **Exclure les gros dossiers** : node_modules, .git, dist, build
4. **Redémarrer les processus** après modification : IDE, serveurs de dev
5. **Documenter** : notez les valeurs choisies

### ❌ À éviter

1. **Ne pas fixer à des valeurs astronomiques** (> 4 millions) sans raison
2. **Ne pas ignorer les erreurs** : si vous atteignez la limite, investiguer pourquoi
3. **Ne pas utiliser le polling** comme solution par défaut (moins performant)
4. **Ne pas oublier de tester après redémarrage**

---

## Conclusion

Augmenter les limites inotify est une configuration essentielle pour tout développeur sous Debian ou Ubuntu. En ajustant les paramètres `max_user_watches`, `max_user_instances` et `max_queued_events` dans `/etc/sysctl.d/60-inotify.conf`, vous éviterez les erreurs frustrantes et améliorerez l'expérience de développement.

**Récapitulatif des commandes clés** :

```bash
# Vérifier les limites actuelles
sysctl fs.inotify

# Configuration permanente
sudo nano /etc/sysctl.d/60-inotify.conf

# Appliquer immédiatement
sudo sysctl -p /etc/sysctl.d/60-inotify.conf

# Diagnostiquer la consommation
for pid in $(ps -ef | awk '{print $2}'); do
    count=$(find /proc/$pid/fd -lname 'anon_inode:inotify' 2>/dev/null | wc -l)
    if [ $count -gt 0 ]; then
        echo "$count watchers: $(ps -p $pid -o comm=)"
    fi
done | sort -rn
```

**Points clés à retenir** :
- inotify surveille les modifications du système de fichiers
- Les limites par défaut sont souvent insuffisantes pour le développement moderne
- Utilisez `/etc/sysctl.d/60-inotify.conf` pour des modifications permanentes
- Chaque watcher consomme ~1 Ko RAM (négligeable sur machines modernes)

---

## Pour aller plus loin

- [Documentation officielle inotify](https://man7.org/linux/man-pages/man7/inotify.7.html)
- [Kernel.org - inotify parameters](https://www.kernel.org/doc/Documentation/sysctl/fs.txt)
- [Chercher dans le code rapidement avec ripgrep]({% post_url 2026-02-16-Chercher-dans-le-code-rapidement-avec-ripgrep %})
- [Optimiser les performances de votre IDE]({% post_url 2025-12-05-Optimiser-les-performances-de-votre-IDE %})
- [Docker et les volumes : guide complet]({% post_url 2025-11-20-Docker-et-les-volumes-guide-complet %})
