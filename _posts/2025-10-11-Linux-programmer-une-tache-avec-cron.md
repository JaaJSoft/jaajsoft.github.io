---
layout: article
title: "Linux : Programmer une tâche avec cron"
tags:
  - linux
  - cron
  - ubuntu
  - debian
  - serveur
author: Pierre Chopinet
---

Pour lancer une commande ou un script tous les jours, toutes les 5 minutes ou au démarrage d'un serveur, l'outil standard sous Linux est cron. Dans ce tutoriel, nous allons voir comment écrire une crontab, dans quel environnement tournent les tâches (c'est là que se cachent la plupart des pièges) et comment trouver pourquoi une tâche ne se lance pas.
<!--more-->

Dans cet article :
- Créer sa première tâche
- La syntaxe d'une crontab
- Gérer sa crontab
- L'environnement des tâches
- Éviter qu'une tâche tourne en double
- Exemples courants
- Crontab système et dossiers cron.daily
- Pourquoi ma tâche ne s'exécute pas ?
- Les timers systemd

Pré-requis : un système Linux avec le service cron et un accès shell. Les comportements décrits sont ceux du cron de Debian et d'Ubuntu (paquet `cron`, testé sur Ubuntu 24.04) ; RHEL et Fedora utilisent cronie, dont le service s'appelle `crond`.

## Créer sa première tâche

On édite sa crontab avec :

```bash
crontab -e
```

La première fois, la commande peut demander quel éditeur utiliser (nano, vim...). Chaque ligne du fichier décrit une tâche. Pour lancer un script de sauvegarde tous les jours à 2 h du matin :

```
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
```

La fin de la ligne envoie la sortie et les erreurs du script dans `/var/log/backup.log`, ce qui permet de savoir ce qui s'est passé si la tâche échoue.

Attention, `crontab -e` modifie la crontab de l'utilisateur courant et la tâche tourne avec ses droits. Or un utilisateur normal ne peut pas écrire dans `/var/log` : cette ligne, comme les autres exemples de l'article qui y écrivent, va dans la crontab de root (`sudo crontab -e`). Dans votre propre crontab, écrivez les journaux dans votre répertoire personnel.

On vérifie ensuite ce qui est programmé :

```bash
crontab -l
```

## La syntaxe d'une crontab

Une ligne de crontab utilisateur contient cinq champs pour la date et l'heure, suivis de la commande :

```
min  heure  jour  mois  jour_sem  commande
0-59 0-23   1-31  1-12  0-7       ...
```

Pour le jour de la semaine, 0 et 7 désignent tous les deux le dimanche, 1 le lundi et 6 le samedi. Les mois et les jours peuvent aussi s'écrire avec les trois premières lettres de leur nom anglais (`jan`, `mon`...). Dans chaque champ, on peut utiliser :

- `*` : toutes les valeurs ;
- `*/5` : toutes les 5 unités (toutes les 5 minutes dans le premier champ) ;
- `1,15` : une liste de valeurs ;
- `1-5` : une plage (du lundi au vendredi dans le dernier champ).

Quelques exemples :

- toutes les 5 minutes : `*/5 * * * * /path/script.sh`
- tous les jours à 2 h : `0 2 * * * /path/script.sh`
- chaque lundi à 9 h : `0 9 * * 1 /path/script.sh`
- le 1er de chaque mois à 6 h : `0 6 1 * * /path/script.sh`
- au démarrage de la machine : `@reboot /path/script.sh`

Les raccourcis `@hourly`, `@daily`, `@weekly`, `@monthly` et `@yearly` remplacent les cinq champs : `@daily`, par exemple, équivaut à `0 0 * * *`. `@reboot` lance la commande une seule fois, au démarrage de la machine : redémarrer le service cron ne la relance pas.

Attention au jour du mois et au jour de la semaine. Quand les deux champs sont renseignés (aucun des deux ne vaut `*`), cron lance la tâche si l'un ou l'autre correspond : `30 4 1,15 * 5` s'exécute à 4 h 30 le 1er et le 15 de chaque mois, mais aussi tous les vendredis. Pour « le premier lundi du mois », on limite le jour du mois aux sept premiers jours et on teste le jour de la semaine dans la commande (`date +%u` donne 1 pour lundi, 7 pour dimanche) :

```
0 9 1-7 * * test $(date +\%u) -eq 1 && /usr/local/bin/rapport.sh
```

Le `\%` de cette ligne n'est pas une coquille : cron remplace chaque `%` non échappé par un retour à la ligne et envoie la suite sur l'entrée standard de la commande. Une commande qui contient un `%` est donc coupée et échoue, souvent sans laisser de trace :

```
# À éviter : la commande est coupée au premier %
0 3 * * * tar czf /backups/www-$(date +%F).tar.gz /var/www
# Correct :
0 3 * * * tar czf /backups/www-$(date +\%F).tar.gz /var/www
```

## Gérer sa crontab

```bash
crontab -e                # éditer
crontab -l                # afficher
crontab -r                # supprimer toute la crontab, sans confirmation
crontab -i -r             # idem, mais en demandant confirmation
sudo crontab -u alice -l  # afficher la crontab d'un autre utilisateur
```

Pour désactiver une tâche temporairement, il suffit de commenter sa ligne avec un `#`.

Les crontabs sont stockées dans `/var/spool/cron/crontabs/`, mais on ne modifie jamais ces fichiers directement : la commande `crontab` vérifie la syntaxe avant de les installer (elle refuse par exemple un fichier dont la dernière ligne ne se termine pas par un retour à la ligne). Inutile ensuite de redémarrer cron, il recharge tout seul les crontabs modifiées.

## L'environnement des tâches

Une tâche cron ne tourne pas dans votre session. Ni `~/.bashrc` ni `~/.profile` ne sont lus, la commande est exécutée par `/bin/sh` (dash sur Debian et Ubuntu, pas bash) et cron ne définit que quelques variables. Pour voir ce que reçoit une tâche, on peut en programmer une qui écrit son environnement dans un fichier :

```
* * * * * env > /tmp/env-cron.txt
```

Avec la crontab de root et le PATH par défaut de Debian, on obtient quelque chose comme :

```
HOME=/root
LOGNAME=root
PATH=/usr/bin:/bin
LANG=C.UTF-8
SHELL=/bin/sh
PWD=/root
```

`HOME` et `LOGNAME` viennent de `/etc/passwd`, `SHELL` vaut `/bin/sh`. Le `PATH` dépend de la distribution : Debian le fixe à `/usr/bin:/bin`, alors qu'Ubuntu, depuis la version 23.04, lance cron avec l'option `-P` et les tâches reçoivent le PATH défini dans `/etc/environment`, plus complet (il contient notamment `/usr/local/bin`, `/usr/sbin` et `/snap/bin`). Dans les deux cas, ce que vous ajoutez au PATH dans votre `~/.bashrc` (`~/.local/bin`, un venv activé...) n'existe pas pour cron.

Le plus sûr est donc d'utiliser des chemins absolus partout : `/usr/bin/python3 /home/user/mon_script.py` plutôt que `python ./mon_script.py`. On peut aussi définir le shell et le PATH en tête de la crontab :

```
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

Attention, cron ne remplace pas les variables dans ces lignes : `PATH=$HOME/bin:$PATH` ne fonctionne pas, il faut écrire les chemins en entier.

Pour un script, mettez un shebang en première ligne :

```bash
#!/usr/bin/env bash
# votre script.sh
```

Puis rendez-le exécutable :

```bash
chmod +x /path/script.sh
```

Pour un script Python dans un environnement virtuel, inutile d'essayer d'activer le venv : `/bin/sh` ne connaît pas `source`, et la tâche échoue avec `/bin/sh: 1: source: not found`. On appelle directement l'interpréteur Python du venv, qui trouve tout seul les paquets installés dedans :

```
# Exemple venv Python
*/10 * * * * /home/user/.venv/bin/python /home/user/app/job.py >> /var/log/app/job.log 2>&1
```

cron utilise le fuseau horaire du système, que l'on vérifie avec :

```bash
timedatectl
```

Il n'y a pas de fuseau par utilisateur : une variable `TZ` définie dans la crontab change l'heure vue par les commandes, pas l'heure à laquelle cron les lance.

Enfin, la sortie d'une tâche est envoyée par mail au propriétaire de la crontab, ou à l'adresse de la variable `MAILTO` (`MAILTO=""` désactive l'envoi). Sans serveur de mail sur la machine, elle est perdue et le journal de cron indique `No MTA installed, discarding output` : encore une raison de rediriger la sortie vers un fichier.

## Éviter qu'une tâche tourne en double

Si une tâche lancée toutes les 5 minutes dure parfois plus de 5 minutes, deux exécutions finissent par se chevaucher. `flock` règle le problème :

```bash
*/5 * * * * flock -n /tmp/monjob.lock -c "/path/script_long.sh" >> /var/log/monjob.log 2>&1
```

`flock` prend un verrou sur le fichier `/tmp/monjob.lock` pendant toute l'exécution du script. Avec `-n`, si l'exécution précédente tient encore le verrou, la nouvelle s'arrête immédiatement (code de retour 1) au lieu d'attendre son tour.

## Exemples courants

Supprimer chaque nuit les fichiers temporaires qui n'ont pas été modifiés depuis plus d'une semaine :

```
30 1 * * * find /var/tmp/app -type f -mtime +7 -delete
```

`find` arrondit l'âge des fichiers au jour inférieur : `-mtime +7` ne retient en fait que les fichiers vieux d'au moins 8 jours.

Sauvegarder une base PostgreSQL tous les jours à 3 h 15, dans un fichier daté :

```
15 3 * * * pg_dump -h 127.0.0.1 -U app -d appdb -F c -f /backups/appdb-$(date +\%F).dump
```

Le mot de passe n'a rien à faire dans la crontab : la documentation de PostgreSQL déconseille la variable `PGPASSWORD` et recommande un fichier `~/.pgpass` dans le répertoire de l'utilisateur qui lance la tâche (une ligne `127.0.0.1:5432:appdb:app:motdepasse`, droits 600 obligatoires).

Relancer un service 30 secondes après le démarrage, dans la crontab de root puisque `systemctl restart` demande les droits root :

```
@reboot sleep 30 && systemctl restart mon-service
```

Le `sleep` n'est pas là par hasard : `@reboot` s'exécute dès le démarrage de cron, parfois avant que les services dont dépend la commande soient prêts.

Vérifier toutes les 5 minutes qu'un service web répond (`-f` fait échouer `curl` quand le serveur renvoie une erreur HTTP) :

```
*/5 * * * * curl -fsS https://status.exemple.com/ping || echo "ping KO" >> /var/log/healthcheck.log
```

Pour qu'une tâche lourde gêne moins le reste de la machine, on baisse sa priorité processeur avec `nice` et sa priorité disque avec `ionice -c3` (classe « idle »). Ce dernier n'a d'effet qu'avec les ordonnanceurs d'entrées/sorties qui gèrent les priorités, bfq et mq-deadline :

```
0 2 * * * nice -n 19 ionice -c3 /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
```

## Crontab système et dossiers cron.daily

En plus des crontabs des utilisateurs, cron lit `/etc/crontab` et les fichiers du dossier `/etc/cron.d/`. Leur format a un champ de plus : l'utilisateur qui exécute la commande, placé entre l'horaire et la commande.

```
# m h dom mon dow user  command
0 2 * * * root /usr/local/bin/backup.sh
```

Ces fichiers s'éditent directement, sans passer par la commande `crontab`, et cron les relit tout seul. Ils doivent appartenir à root et ne pas être modifiables par le groupe ou les autres utilisateurs. Ceux de `/etc/cron.d/` doivent en plus avoir un nom composé uniquement de lettres, de chiffres, de tirets et de soulignés : un fichier `sauvegarde.conf` ou `backup.cron` est tout simplement ignoré.

Les dossiers `/etc/cron.hourly`, `/etc/cron.daily`, `/etc/cron.weekly` et `/etc/cron.monthly` contiennent des scripts lancés par `run-parts` depuis `/etc/crontab`. Les mêmes règles de nommage s'appliquent, et les scripts doivent être exécutables : un script `backup.sh` déposé dans `/etc/cron.daily` ne sera jamais lancé, il faut l'appeler `backup`. Pour savoir ce qui sera réellement exécuté :

```bash
run-parts --test /etc/cron.daily
```

Sur une machine qui n'est pas allumée en permanence, comme un portable, anacron rattrape les tâches quotidiennes, hebdomadaires et mensuelles qui n'ont pas pu tourner à l'heure prévue. Quand il est installé, `/etc/crontab` lui laisse d'ailleurs la gestion de ces trois dossiers.

## Pourquoi ma tâche ne s'exécute pas ?

Commencez par les journaux, cron y note le lancement de chaque tâche :

```bash
# via journald
journalctl -u cron -f
# ou dans syslog, si rsyslog est installé (c'est le cas par défaut sur Ubuntu)
sudo grep CRON /var/log/syslog
```

Si la tâche n'y apparaît pas du tout, vérifiez la syntaxe de l'horaire, le nom du fichier s'il est dans `/etc/cron.d/`, et que le service tourne avec `systemctl status cron` (`crond` sur RHEL et Fedora).

Si elle apparaît mais ne fait pas ce qu'on attend, le problème vient presque toujours de ce qui a été vu plus haut : commande introuvable à cause du PATH, `%` non échappé, droits insuffisants pour écrire un fichier, script non exécutable. La redirection de la sortie vers un fichier (`>> fichier.log 2>&1`) donne alors le message d'erreur exact.

## Les timers systemd

Sur les distributions qui utilisent systemd, les timers sont une alternative à cron. Une tâche y est décrite par deux unités, un service et un timer, ce qui donne accès aux dépendances entre services, aux journaux de `journalctl`, aux limites de ressources (`MemoryMax=`, `CPUQuota=`) et aux options de sandboxing de systemd. Avec `Persistent=true`, un timer lance au démarrage les exécutions manquées pendant que la machine était éteinte, à la manière d'anacron. `systemctl list-timers` affiche les timers actifs, dont ceux du système comme `apt-daily.timer`.

Pour une simple commande à heure fixe, cron reste plus rapide à mettre en place.

## Voir aussi

- [Kubernetes : Programmer des tâches avec CronJob]({% post_url 2025-10-12-Kubernetes-programmer-une-tache-avec-cronjob %})
- [free : surveiller et comprendre l'utilisation mémoire sous Linux]({% post_url 2026-03-16-Surveiller-la-memoire-avec-free %})
- [df/du : surveiller et analyser l'espace disque sous Linux]({% post_url 2026-03-01-Surveiller-espace-disque-avec-df-et-du %})
- [Activer les mises à jour de sécurité automatiques sur Ubuntu/Debian]({% post_url 2025-12-19-Activer-les-mises-a-jour-de-securite-automatiques-sur-Ubuntu-Debian %})
- [Page de manuel crontab(5) (Ubuntu 24.04)](https://manpages.ubuntu.com/manpages/noble/en/man5/crontab.5.html)
- [Page de manuel cron(8) (Ubuntu 24.04)](https://manpages.ubuntu.com/manpages/noble/en/man8/cron.8.html)
- [Debian Wiki : cron](https://wiki.debian.org/cron)
- [Ubuntu : CronHowto](https://help.ubuntu.com/community/CronHowto)
