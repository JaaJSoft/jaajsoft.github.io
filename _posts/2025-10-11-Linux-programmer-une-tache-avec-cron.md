---
layout: article
title: "Linux : Programmer une tâche avec cron"
author: Pierre Chopinet
tags:
  - linux
  - cron
  - ubuntu
  - debian
  - serveur
---

Besoin d’automatiser une commande ou un script tous les jours, toutes les 5 minutes ou au démarrage du serveur ? `cron` est l’outil standard sous Linux pour planifier des tâches récurrentes.
Dans ce guide pratique, on va voir comment créer sa première tâche en moins de 5 minutes, comprendre la syntaxe, éviter les pièges courants et dépanner un cron s'il ne tourne pas.
<!--more-->

---

## 1) L’essentiel

- Éditer votre crontab utilisateur :

```bash
crontab -e
```

- Ajouter une ligne « tous les jours à 2h » :

```
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
```

- Vérifier ce qui est programmé :

```bash
crontab -l
```

- Voir les logs (Ubuntu/Debian) :

```bash
# via journald
journalctl -u cron -f
# ou dans syslog
sudo grep CRON /var/log/syslog
```

---

## 2) Comprendre la syntaxe d’une crontab

Une ligne de crontab (utilisateur) a 5 champs de date/heure suivis de la commande :

```
min  heure  jour  mois  jour_sem  commande
0-59 0-23   1-31  1-12  0-7       ...
```

- `jour_sem` : 0 ou 7 = dimanche, 1 = lundi, … 6 = samedi
- Astuce : `*` = « toutes les valeurs », `*/5` = « toutes les 5 unités », `1,15` = « les 1 et 15 », `1-5` = « de 1 à 5 »

Exemples rapides :

- Toutes les 5 minutes : `*/5 * * * * /path/script.sh`
- Tous les jours à 02:00 : `0 2 * * * /path/script.sh`
- Chaque lundi à 09:00 : `0 9 * * 1 /path/script.sh`
- Le 1er de chaque mois à 06:00 : `0 6 1 * * /path/script.sh`
- Au démarrage de la machine : `@reboot /path/script.sh`

Raccourcis utiles :

- `@hourly`, `@daily`, `@weekly`, `@monthly`, `@yearly`, `@reboot`

> Attention : la crontab système (`/etc/crontab` ou fichiers dans `/etc/cron.d/`) ajoute un champ « utilisateur » après les 5 champs temps. Exemple : `0 2 * * * root /path/script.sh`.

---

## 3) Créer/éditer votre crontab utilisateur

- Ouvrir : `crontab -e` (première fois, il peut vous demander l’éditeur : nano/vi)
- Lister : `crontab -l`
- Supprimer tout : `crontab -r` (dangereux, demandez confirmation avec `-i`)

Vos tâches tourneront avec vos droits utilisateur, via le service `cron` du système.

---

## 4) Bonnes pratiques

- Chemins absolus uniquement : `/usr/bin/python` et `/home/user/mon_script.py`, pas `python` ni `./mon_script.py`.
- Définir un PATH et le shell en tête de la crontab si nécessaire :

```
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

- Rediriger la sortie et les erreurs pour diagnostiquer :

```
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
```

- Scripts exécutables + shebang :

```bash
#!/usr/bin/env bash
# votre script.sh
```

Puis :

```bash
chmod +x /path/script.sh
```

- Éviter les chevauchements : utilisez `flock` pour qu’une tâche ne tourne pas en double si la précédente n’est pas finie :

```bash
*/5 * * * * flock -n /tmp/monjob.lock -c "/path/script_long.sh" >> /var/log/monjob.log 2>&1
```

- Variables d’environnement : cron a un environnement minimal. Exportez ce dont vous avez besoin, ou sourcez votre venv :

```
# Exemple venv Python
*/10 * * * * source /home/user/.venv/bin/activate && python /home/user/app/job.py >> /var/log/app/job.log 2>&1
```

- Fuseau horaire : cron utilise le fuseau du système. Vérifiez :

```bash
timedatectl
```

---

## 5) Exemples courants

- Sauvegarde quotidienne à 2h :

```
0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
```

- Nettoyage des fichiers temporaires vieux de 7 jours (toutes les nuits) :

```
30 1 * * * find /var/tmp/app -type f -mtime +7 -delete
```

- Dump d’une base PostgreSQL tous les jours à 3h15 :

```
15 3 * * * PGUSER=app PGPASSWORD=secret pg_dump -h 127.0.0.1 -d appdb -F c -f /backups/appdb-$(date +\%F).dump
```

- Relancer un service au boot après 30 secondes :

```
@reboot sleep 30 && systemctl restart mon-service
```

- Ping santé toutes les 5 minutes :

```
*/5 * * * * curl -fsS https://status.exemple.com/ping || echo "ping KO" >> /var/log/healthcheck.log
```

---

## 6) Cron système vs crontab utilisateur

- Crontab utilisateur : `crontab -e` (pas de champ « utilisateur »)
- Crontab système : `/etc/crontab` et `/etc/cron.d/*.conf` (nécessite `root`, contient un champ utilisateur)
- Dossiers périodiques (pilotés par `/etc/crontab` via `run-parts`) : `/etc/cron.hourly`, `/etc/cron.daily`, `/etc/cron.weekly`, `/etc/cron.monthly`
- Machines qui ne tournent pas H24 (laptops) : `anacron` garantit l’exécution « à retardement » des tâches quotidiennes/hebdo/mensuelles.

Exemple d’entrée dans `/etc/crontab` :

```
# m h dom mon dow user  command
0 2 * * * root /usr/local/bin/backup.sh
```

---

## 7) Dépannage : pourquoi ma tâche ne s’exécute pas ?

Vérifiez les points suivants :

- Le script est-il exécutable (`chmod +x`) et a-t-il un shebang correct ?
- Les chemins sont-ils absolus (binaire, fichiers, venv) ?
- Permissions : l’utilisateur de la crontab a-t-il le droit d’exécuter/écrire où il faut ?
- Journaux : voyez `journalctl -u cron` ou `grep CRON /var/log/syslog`.
- Redirection mise en place (`>> ... 2>&1`) pour capturer les erreurs ?
- Fin de ligne : le fichier crontab doit se terminer par un saut de ligne (évite que la dernière ligne soit ignorée).
- Caractères spéciaux (`%` dans crontab) : échappez-les avec `\%` dans les commandes (ex. `date +\%F`).
- Variables non définies : rappelez `PATH`, `SHELL`, variables d’appli.
- Système différent : sur certains systèmes, le service s’appelle `crond` (ex. RHEL), adaptez `systemctl status crond`.

---

## 8) Alternatives modernes: systemd timers

Sur des distributions modernes, les « timers » systemd offrent plus de contrôle (dépendances, sandboxing, calendriers `OnCalendar=`).
Pour des besoins complexes, envisagez un service + timer systemd. Pour des tâches simples, `cron` reste parfait.

---

## FAQ

- Comment exécuter un script Python avec son venv ?
  - Utilisez le Python du venv en chemin absolu : `/home/user/.venv/bin/python /home/user/app/job.py`.
- Puis-je limiter l’usage CPU/RAM ?
  - Enveloppez la commande avec `systemd-run --scope -p MemoryMax=... -p CPUQuota=...` ou utilisez `nice`/`ionice`.
- Comment arrêter temporairement une tâche ?
  - Commentez la ligne dans `crontab -e` en la préfixant avec `#`.

---

## Voir aussi

- [Activer les mises à jour de sécurité automatiques sur Ubuntu/Debian]({% post_url 2025-12-19-Activer-les-mises-a-jour-de-securite-automatiques-sur-Ubuntu-Debian %})
- [Installer et configurer Fail2ban sur un serveur Ubuntu/Debian]({% post_url 2025-09-21-Installer-et-configurer-Fail2ban-sur-Ubuntu-Debian %})
- [Kubernetes : Programmer des tâches avec CronJob]({% post_url 2025-10-12-Kubernetes-programmer-une-tache-avec-cronjob %})

## Références

- [man 5 crontab (format)](https://man7.org/linux/man-pages/man5/crontab.5.html) et [man 8 cron (service)](https://man7.org/linux/man-pages/man8/cron.8.html)
- [Debian Wiki – Cron](https://wiki.debian.org/cron)
- [Ubuntu CronHowto](https://help.ubuntu.com/community/CronHowto)
