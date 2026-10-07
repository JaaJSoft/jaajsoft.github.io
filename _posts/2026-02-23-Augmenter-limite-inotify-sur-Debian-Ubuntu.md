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

Lorsque vous développez sur Linux avec des outils modernes (IDE, hot-reload, watchers), vous rencontrez souvent l'erreur "too many open files" ou "inotify watch limit reached". Ces messages indiquent que la limite par défaut d'inotify est atteinte. Ce guide vous montre comment augmenter ces limites sur Debian et Ubuntu.
<!--more-->

Dans cet article, vous découvrirez :
- Ce qu'est inotify et pourquoi les limites existent
- Comment vérifier vos limites actuelles
- Comment augmenter temporairement les limites
- Comment rendre les modifications permanentes
- Les valeurs recommandées selon votre usage
- Comment diagnostiquer les processus qui consomment des watchers

Pré-requis : Debian ou Ubuntu (la procédure s'applique aussi aux autres distributions Linux modernes) et accès `sudo`.

---

## Qu'est-ce qu'inotify ?

`inotify` (inode notify) est un sous-système du noyau Linux qui surveille les modifications du système de fichiers. Il permet aux applications de recevoir des notifications lorsqu'un fichier ou répertoire est créé, modifié, supprimé ou déplacé.

---

## Les trois limites d'inotify

Le système inotify possède trois paramètres configurables :

### max_user_watches

Nombre maximum de fichiers et répertoires surveillés par utilisateur.

```bash
cat /proc/sys/fs/inotify/max_user_watches
# Par défaut : 8192 ou 524288 (selon distribution)
```

> Note : depuis le noyau Linux 5.11, cette valeur par défaut n'est plus figée. Le noyau la dimensionne dynamiquement pour ne pas dépasser environ 1 % de la mémoire adressable, dans une plage de 8192 à 1048576 (comportement calqué sur `epoll`). Sur les noyaux plus anciens, la valeur historique de 8192 est souvent trop basse.

Problème courant : dépassement lors du watch de gros projets (node_modules, monorepos).

### max_user_instances

Nombre maximum d'instances inotify par utilisateur (nombre de processus pouvant surveiller des fichiers).

```bash
cat /proc/sys/fs/inotify/max_user_instances
# Par défaut : 128
```

Problème courant : trop d'outils tournent simultanément (IDE + webpack + tests + docker).

### max_queued_events

Nombre maximum d'événements en attente dans la file avant qu'ils ne soient traités.

```bash
cat /proc/sys/fs/inotify/max_queued_events
# Par défaut : 16384
```

Problème courant : modifications massives (git checkout, npm install) peuvent saturer la file.

---

## Vérifier vos limites actuelles

### Afficher toutes les limites

```bash
sysctl fs.inotify
```

Sortie typique :
```
fs.inotify.max_queued_events = 16384
fs.inotify.max_user_instances = 128
fs.inotify.max_user_watches = 524288
```

### Vérifier combien d'instances inotify sont ouvertes

```bash
find /proc/*/fd -lname 'anon_inode:inotify' 2>/dev/null | wc -l
```

Attention à ne pas confondre les deux notions. Cette commande compte les **instances** inotify (un descripteur de fichier `anon_inode:inotify` par appel à `inotify_init`), pas les **watches**. Une seule instance peut poser des milliers de watches ; c'est `max_user_watches` qui limite le nombre total de watches, `max_user_instances` le nombre d'instances. Pour compter les watches réellement posés, additionnez les lignes `inotify wd:` des fichiers `/proc/*/fdinfo/*` :

```bash
find /proc/*/fdinfo -type f 2>/dev/null | xargs grep -c '^inotify' 2>/dev/null | awk -F: '{sum+=$NF} END {print sum}'
```

### Identifier les processus consommateurs

```bash
for pid in $(ps -ef | awk '{print $2}'); do
    count=$(find /proc/$pid/fd -lname 'anon_inode:inotify' 2>/dev/null | wc -l)
    if [ $count -gt 0 ]; then
        echo "$count instances inotify: $(ps -p $pid -o comm=)"
    fi
done | sort -rn
```

Cette commande liste les processus avec leurs nombres de watchers (utile pour diagnostiquer).

---

## Augmenter les limites temporairement

### Modifier pour la session en cours

```bash
sudo sysctl fs.inotify.max_user_watches=524288
sudo sysctl fs.inotify.max_user_instances=512
sudo sysctl fs.inotify.max_queued_events=32768
```

> Limitation : ces changements sont perdus au redémarrage.

Usage : test rapide ou débogage ponctuel.

---

## Augmenter les limites de façon permanente

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

Résultat attendu :
```
fs.inotify.max_user_watches = 524288
```

---

## Impact mémoire des limites

Chaque watch **effectivement posé** consomme de l'ordre de 1 Ko de mémoire noyau (non paginable). Rien n'est préalloué : relever la limite n'immobilise aucune RAM tant que les watches ne sont pas réellement créés.

Le calcul suivant donne donc le **plafond théorique**, atteint uniquement si les 524 288 watches sont tous utilisés simultanément :
```
524288 watches × 1 Ko = 512 Mo (maximum théorique)
```

En pratique, un poste de développement en utilise une petite fraction. Sur une machine moderne (16 Go de RAM ou plus), même le pire cas reste raisonnable.

> Conseil : ne dépassez pas 4 millions de watchers sauf cas extrême (risque de ralentissements).

---

## Debugging : lister les watchers actifs

### Avec inotify-tools

Installez le paquet :
```bash
sudo apt update
sudo apt install inotify-tools
```

Surveillez en temps réel :
```bash
inotifywatch -v /chemin/vers/dossier
```

### Avec un script

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

Sortie exemple :
```
  45 node
  12 java
   8 code
   3 docker
```

---

## Bonnes pratiques

### À faire

- **Augmenter progressivement** : commencez par 524288, augmentez si nécessaire
- **Monitorer la RAM** : surveillez l'impact mémoire avec `htop` ou `free -h`
- **Exclure les gros dossiers** : node_modules, .git, dist, build
- **Redémarrer les processus** après modification : IDE, serveurs de dev
- **Documenter** : notez les valeurs choisies

### À éviter

- **Ne pas fixer à des valeurs astronomiques** (> 4 millions) sans raison
- **Ne pas ignorer les erreurs** : si vous atteignez la limite, investiguer pourquoi
- **Ne pas utiliser le polling** comme solution par défaut (moins performant)
- **Ne pas oublier de tester après redémarrage**

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
        echo "$count instances inotify: $(ps -p $pid -o comm=)"
    fi
done | sort -rn
```

**Points clés à retenir** :
- inotify surveille les modifications du système de fichiers
- Les limites par défaut sont souvent insuffisantes pour le développement moderne
- Utilisez `/etc/sysctl.d/60-inotify.conf` pour des modifications permanentes
- Chaque watch réellement posé consomme environ 1 Ko de RAM noyau, sans préallocation (la limite est un plafond, pas une réservation)

---

## Pour aller plus loin

- [Documentation officielle inotify](https://man7.org/linux/man-pages/man7/inotify.7.html)
- [Kernel.org - inotify parameters](https://www.kernel.org/doc/Documentation/sysctl/fs.txt)

## Voir aussi

- [Chercher dans le code rapidement avec ripgrep]({% post_url 2026-02-16-Chercher-dans-le-code-rapidement-avec-ripgrep %})
- [Sed : éditer des fichiers en ligne de commande avec des regex]({% post_url 2026-01-19-Sed-editer-des-fichiers-en-ligne-de-commande %})
- [Activer les mises à jour de sécurité automatiques sur Ubuntu/Debian]({% post_url 2025-12-19-Activer-les-mises-a-jour-de-securite-automatiques-sur-Ubuntu-Debian %})
- [Surveiller la mémoire avec free]({% post_url 2026-03-16-Surveiller-la-memoire-avec-free %})
- [Surveiller l'espace disque avec df et du]({% post_url 2026-03-01-Surveiller-espace-disque-avec-df-et-du %})
