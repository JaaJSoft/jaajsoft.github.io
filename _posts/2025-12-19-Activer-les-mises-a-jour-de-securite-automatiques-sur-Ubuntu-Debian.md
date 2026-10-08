---
layout: article
title: "Activer les mises à jour de sécurité automatiques sur Ubuntu/Debian"
description: "Activer les mises à jour de sécurité automatiques sur Ubuntu et Debian avec unattended-upgrades : configuration, fréquence, tests, logs et redémarrages."
tags:
  - linux
  - ubuntu
  - debian
  - sécurité
  - administration
author: Pierre Chopinet
---

Sur un serveur, un correctif de sécurité ne sert à rien tant qu'il n'est pas installé, et attendre de penser à lancer `apt upgrade` peut laisser une faille ouverte pendant des semaines. Ubuntu et Debian fournissent pour cela le paquet `unattended-upgrades`, qui installe automatiquement les mises à jour de sécurité tous les jours. Dans ce tutoriel, nous allons voir comment l'activer, choisir ce qu'il installe et vérifier qu'il fait bien son travail.
<!--more-->

Dans cet article :
- Installation et activation
- Comment les mises à jour sont lancées
- Choisir les mises à jour installées
- Exclure certains paquets
- Gérer les redémarrages
- Supprimer les paquets devenus inutiles
- Recevoir un rapport par mail
- Tester la configuration et lire les logs
- Désactiver les mises à jour automatiques

Pré-requis : Ubuntu ou Debian, avec un accès root ou `sudo`. Les commandes et les sorties de l'article ont été testées sur Ubuntu 24.04, avec unattended-upgrades 2.9.1.

## Installation et activation

Sur Ubuntu, `unattended-upgrades` est installé par défaut et les mises à jour de sécurité automatiques sont actives dès l'installation du système. Pour vérifier que le paquet est bien là :

```bash
dpkg -l unattended-upgrades
```

La dernière ligne doit commencer par `ii`, ce qui signifie que le paquet est installé :

```
ii  unattended-upgrades 2.9.1+nmu4ubuntu1 all          automatic installation of security upgrades
```

Sinon, on l'installe. Sur Debian, on peut y ajouter `apt-listchanges`, qui permet de recevoir par mail les nouveautés importantes des paquets mis à jour (nous y reviendrons) :

```bash
sudo apt update
sudo apt install unattended-upgrades apt-listchanges
```

À l'installation, le paquet active directement les mises à jour automatiques. Pour les activer (ou les désactiver) par la suite, on relance sa configuration :

```bash
sudo dpkg-reconfigure -plow unattended-upgrades
```

L'option `-plow` demande d'afficher les questions de priorité basse, comme celle de ce paquet. `dpkg-reconfigure` le fait déjà par défaut, mais c'est la commande que donne le README du projet. Répondez *Yes* à la question "Automatically download and install stable updates?". La commande écrit alors le fichier `/etc/apt/apt.conf.d/20auto-upgrades` :

```conf
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

La première ligne met à jour la liste des paquets, comme un `apt update`, et la seconde lance `unattended-upgrades`. La valeur est un intervalle en jours : `1` pour tous les jours, `2` pour tous les deux jours, `0` pour désactiver. Pour voir la configuration réellement prise en compte par APT, tous fichiers confondus, on utilise `apt-config` :

```bash
apt-config dump APT::Periodic
```

```
APT::Periodic "";
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

Le même fichier accepte d'autres options. `APT::Periodic::AutocleanInterval "7";` vide chaque semaine le cache des paquets téléchargés qui ne sont plus disponibles dans les dépôts, comme un `apt-get autoclean`. `APT::Periodic::Download-Upgradeable-Packages "1";` télécharge chaque jour toutes les mises à jour disponibles sans les installer, ce qui accélère un `apt upgrade` manuel.

## Comment les mises à jour sont lancées

Avec systemd, ce sont deux timers d'APT qui font le travail :

- `apt-daily.timer` met à jour la liste des paquets et télécharge les mises à jour, à 6 h et à 18 h (avec un délai aléatoire d'au plus 12 heures) ;
- `apt-daily-upgrade.timer` installe les mises à jour avec `unattended-upgrades` et nettoie le cache, à 6 h (avec un délai aléatoire d'au plus une heure).

Les délais aléatoires évitent que toutes les machines interrogent les miroirs en même temps. Les deux timers lancent le script `/usr/lib/apt/apt.systemd.daily`, qui lit les options `APT::Periodic` vues plus haut. Pour voir leur prochaine exécution (colonne `NEXT`) et la dernière (colonne `LAST`) :

```bash
systemctl list-timers apt-daily.timer apt-daily-upgrade.timer
```

Les timers ont l'option `Persistent=true` : si la machine était éteinte à l'heure prévue, la tâche est lancée au démarrage suivant. C'est pour cela que `unattended-upgrades` tourne parfois juste après le boot, et qu'un `apt install` lancé à ce moment doit attendre qu'il ait fini.

Attention, le service `unattended-upgrades.service` ne lance pas les mises à jour, comme le rappelle sa description ("Unattended Upgrades Shutdown"). Son statut ne dit donc rien sur les mises à jour quotidiennes. Il ne sert qu'à l'extinction de la machine : si une mise à jour est en cours, il demande à `unattended-upgrades` de s'arrêter et attend que celui-ci ait terminé. Cela fonctionne grâce à l'option `MinimalSteps`, active par défaut, qui fait installer les paquets par petits groupes pour pouvoir s'arrêter proprement entre deux. Et si une installation a quand même été interrompue, `unattended-upgrades` relance `dpkg --force-confold --configure -a` à son passage suivant (option `AutoFixInterruptedDpkg`, elle aussi active par défaut).

Sur une machine sans systemd, c'est le script `/etc/cron.daily/apt-compat` qui prend le relais : il ne fait rien si systemd tourne, et lance sinon le même script `apt.systemd.daily`.

## Choisir les mises à jour installées

Tous les autres réglages se trouvent dans `/etc/apt/apt.conf.d/50unattended-upgrades`. Les lignes qui commencent par `//` sont des commentaires, et la plupart des options y figurent déjà, commentées, avec leur valeur par défaut.

Le README du projet conseille de ne pas modifier ce fichier et de mettre vos réglages dans un fichier lu après lui, par exemple `/etc/apt/apt.conf.d/52unattended-upgrades-local`. Sinon, une nouvelle version du fichier livrée avec le paquet peut entrer en conflit avec vos modifications et bloquer la mise à jour d'`unattended-upgrades` lui-même. Les deux méthodes fonctionnent, et les exemples qui suivent peuvent aller dans l'un ou l'autre fichier.

Sur Ubuntu, les dépôts autorisés sont listés dans `Allowed-Origins`. Voici la liste par défaut (sans ses commentaires) :

```conf
Unattended-Upgrade::Allowed-Origins {
	"${distro_id}:${distro_codename}";
	"${distro_id}:${distro_codename}-security";
	"${distro_id}ESMApps:${distro_codename}-apps-security";
	"${distro_id}ESM:${distro_codename}-infra-security";
//	"${distro_id}:${distro_codename}-updates";
//	"${distro_id}:${distro_codename}-proposed";
//	"${distro_id}:${distro_codename}-backports";
};
```

`${distro_id}` et `${distro_codename}` sont remplacés par le nom de la distribution et le nom de code de sa version : `Ubuntu` et `noble` pour Ubuntu 24.04. La première ligne autorise le dépôt principal de la version : d'après le commentaire du fichier, une mise à jour de sécurité peut avoir besoin d'une nouvelle dépendance qui se trouve dans ce dépôt. Viennent ensuite les mises à jour de sécurité, puis celles de l'ESM (*Expanded Security Maintenance*), disponibles avec Ubuntu Pro. Les lignes commentées correspondent aux mises à jour recommandées (`-updates`), qui corrigent des bugs sans lien avec la sécurité, aux paquets encore en test (`-proposed`, à ne pas activer sur un serveur) et aux backports (`-backports`).

Pour installer aussi les corrections de bugs (ce que je déconseille en production), il suffit de décommenter la ligne `-updates`, ou de l'ajouter dans `52unattended-upgrades-local` :

```conf
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-updates";
};
```

Dans un fichier séparé, les listes s'ajoutent à celles du fichier d'origine, ce que l'on peut vérifier avec `apt-config dump Unattended-Upgrade::Allowed-Origins` :

```
Unattended-Upgrade::Allowed-Origins "";
Unattended-Upgrade::Allowed-Origins:: "${distro_id}:${distro_codename}";
Unattended-Upgrade::Allowed-Origins:: "${distro_id}:${distro_codename}-security";
Unattended-Upgrade::Allowed-Origins:: "${distro_id}ESMApps:${distro_codename}-apps-security";
Unattended-Upgrade::Allowed-Origins:: "${distro_id}ESM:${distro_codename}-infra-security";
Unattended-Upgrade::Allowed-Origins:: "${distro_id}:${distro_codename}-updates";
```

Pour remplacer complètement la liste, il faut d'abord la vider avec `#clear` :

```conf
#clear Unattended-Upgrade::Allowed-Origins;
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
};
```

Sur Debian, la liste s'appelle `Origins-Pattern` et utilise une autre syntaxe. Voici ses lignes actives par défaut :

```conf
Unattended-Upgrade::Origins-Pattern {
        "origin=Debian,codename=${distro_codename},label=Debian";
        "origin=Debian,codename=${distro_codename},label=Debian-Security";
        "origin=Debian,codename=${distro_codename}-security,label=Debian-Security";
};
```

La première ligne correspond au dépôt principal de la version, qui reçoit les mises à jour de chaque version intermédiaire de Debian. Contrairement à Ubuntu, la configuration par défaut de Debian installe donc les mises à jour de la version stable en plus des correctifs de sécurité, comme l'indique le README du projet. Les deux lignes suivantes correspondent au dépôt de sécurité, avec son ancien nom (jusqu'à Debian 10) et son nom actuel (`bookworm-security` par exemple, depuis Debian 11).

## Exclure certains paquets

`Package-Blacklist` permet d'exclure des paquets, par exemple une base de données que l'on préfère mettre à jour soi-même, au moment choisi :

```conf
Unattended-Upgrade::Package-Blacklist {
    "mysql-server";    // mysql-server, mysql-server-8.0, mysql-server-core-8.0...
    "postgresql-";     // tous les paquets qui commencent par postgresql-
    "nginx$";          // seulement le paquet nginx
};
```

Attention, ces motifs sont des expressions régulières Python, comparées au début du nom des paquets, et pas des jokers du shell. `"nginx"` exclurait donc aussi `nginx-common` ou `nginx-core` : pour viser un seul paquet, on termine le motif par `$`. De même, dans `"linux-image-*"`, l'étoile porte sur le tiret qui la précède et non sur la suite du nom. Comme la comparaison se fait sur le début du nom, ce motif exclut quand même tous les paquets qui commencent par `linux-image`, mais `"linux-image"` aurait suffi.

Un paquet exclu ne reçoit plus aucune mise à jour automatique, pas même ses correctifs de sécurité : il faudra les installer vous-même. Et d'après le README, si un paquet dépend d'un paquet exclu, aucun des deux n'est mis à jour.

## Gérer les redémarrages

Une mise à jour du noyau ou de la libc ne prend effet qu'après un redémarrage, ce que signale le fichier `/var/run/reboot-required`. Le nom des paquets concernés est ajouté dans `/var/run/reboot-required.pkgs` :

```bash
cat /var/run/reboot-required
cat /var/run/reboot-required.pkgs
```

Si le premier fichier n'existe pas, aucun redémarrage n'est nécessaire. Sur Ubuntu, ce sont les paquets concernés qui le créent, avec le message `*** System restart required ***`, qui s'affiche aussi à la connexion. Sur Debian, c'est `unattended-upgrades` qui le crée, vide, après une mise à jour du noyau.

Pour que la machine redémarre toute seule quand c'est nécessaire :

```conf
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "03:00";
```

Le redémarrage se fait sans confirmation, à la fin de l'exécution d'`unattended-upgrades`, si `/var/run/reboot-required` existe. `Automatic-Reboot-Time` donne l'heure du redémarrage : la valeur est passée telle quelle à la commande `shutdown -r` et vaut `now` par défaut. La machine redémarre même si des utilisateurs sont connectés, sauf avec `Unattended-Upgrade::Automatic-Reboot-WithUsers "false";`.

Un redémarrage automatique coupe les services sans prévenir : c'est à réserver aux machines qui peuvent s'arrêter quelques minutes à 3 h du matin, ou qui sont redondées. Sur les serveurs de production, mieux vaut planifier les redémarrages soi-même.

Toutes les mises à jour ne demandent pas de redémarrer la machine. Par contre, un service qui a chargé une bibliothèque mise à jour continue d'utiliser l'ancienne version tant qu'il n'est pas relancé. Depuis Ubuntu 24.04, le paquet `needrestart`, appelé après les mises à jour, redémarre automatiquement les services concernés.

## Supprimer les paquets devenus inutiles

Trois options gèrent le nettoyage après les mises à jour :

```conf
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
Unattended-Upgrade::Remove-New-Unused-Dependencies "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
```

Les deux premières sont déjà actives par défaut : elles suppriment les anciens noyaux et les dépendances devenues inutiles à cause de la mise à jour. La troisième, désactivée par défaut, supprime toutes les dépendances inutiles, comme un `apt autoremove`.

## Recevoir un rapport par mail

Pour recevoir un mail après les mises à jour :

```conf
Unattended-Upgrade::Mail "admin@example.com";
Unattended-Upgrade::MailReport "on-change";
```

`MailReport` accepte trois valeurs : `always` envoie un mail à chaque exécution, `only-on-error` seulement en cas d'erreur, et `on-change`, la valeur par défaut, seulement s'il s'est passé quelque chose (paquets installés, retenus ou supprimés, ou erreur).

Pour envoyer ses mails, `unattended-upgrades` utilise `/usr/sbin/sendmail`, fourni par un serveur de mail comme Postfix, ou à défaut la commande `mail`. Le plus simple est d'installer Postfix, ainsi que `mailutils` pour avoir la commande `mail` :

```bash
sudo apt install postfix mailutils
```

Pendant l'installation de Postfix, choisissez *Internet Site* et entrez votre nom de domaine, ou *Internet with smarthost* si vos mails doivent passer par un relais SMTP. Testez ensuite l'envoi :

```bash
echo "Test" | mail -s "Test email" admin@example.com
```

Si `apt-listchanges` est installé et que `sendmail` est disponible, `unattended-upgrades` lui fait aussi envoyer par mail les nouveautés importantes des paquets mis à jour (les fichiers `NEWS.Debian`). Le destinataire, `root` par défaut, se règle dans `/etc/apt/listchanges.conf`.

## Tester la configuration et lire les logs

Pas besoin d'attendre le lendemain pour tester sa configuration, on peut lancer `unattended-upgrade` à la main (`unattended-upgrades`, avec un s, est un simple lien vers la même commande) :

```bash
sudo unattended-upgrade --dry-run -v
```

`--dry-run` simule l'exécution sans rien installer (les paquets sont quand même téléchargés) et `-v` affiche les messages d'information. Sur une machine Ubuntu 24.04 qui avait des mises à jour en attente, voici les lignes principales de la sortie (la liste des paquets est raccourcie) :

```
Starting unattended upgrades script
Allowed origins are: o=Ubuntu,a=noble, o=Ubuntu,a=noble-security, o=UbuntuESMApps,a=noble-apps-security, o=UbuntuESM,a=noble-infra-security
Initial blacklist:
Initial whitelist (not strict):
Option --dry-run given, *not* performing real actions
Packages that will be upgraded: fonts-opensymbol libfreetype-dev libfreetype6 [...] sudo uno-libs-private ure
Writing dpkg log to /var/log/unattended-upgrades/unattended-upgrades-dpkg.log
[...]
All upgrades installed
The list of kept packages can't be calculated in dry-run mode.
```

La ligne `Allowed origins are` montre les dépôts autorisés une fois les variables remplacées et `Initial blacklist` les paquets exclus, ce qui permet de vérifier sa configuration. Pour plus de détails, par exemple pour comprendre pourquoi un paquet n'est pas mis à jour, on remplace `-v` par `--debug`.

Chaque exécution, automatique ou manuelle, est journalisée dans `/var/log/unattended-upgrades/`, qui contient trois fichiers :

- `unattended-upgrades.log` reprend les messages ci-dessus, précédés de la date et de l'heure ;
- `unattended-upgrades-dpkg.log` contient la sortie de dpkg pendant l'installation des paquets ;
- `unattended-upgrades-shutdown.log` est le journal du service `unattended-upgrades.service`, qui surveille l'extinction.

Pour un historique de toutes les installations faites par APT, manuelles ou automatiques, il y a aussi `/var/log/apt/history.log`. Attention, un test lancé avec `--dry-run` y apparaît lui aussi, avec une ligne `Upgrade:`, alors que rien n'a été installé. Enfin, si vos logs sont centralisés, `Unattended-Upgrade::SyslogEnable "true";` envoie aussi ces messages à syslog, avec la *facility* `daemon` par défaut (modifiable avec `SyslogFacility`).

## Désactiver les mises à jour automatiques

Le plus simple est de relancer `sudo dpkg-reconfigure -plow unattended-upgrades` et de répondre *No*. La commande remet alors les deux valeurs de `/etc/apt/apt.conf.d/20auto-upgrades` à `0` :

```conf
APT::Periodic::Update-Package-Lists "0";
APT::Periodic::Unattended-Upgrade "0";
```

On peut aussi modifier ces deux lignes à la main. Par contre, désactiver le service `unattended-upgrades` avec `systemctl` ne suffit pas : comme on l'a vu, ce service ne s'occupe que de l'extinction, et ce sont les timers d'APT qui lancent les mises à jour.

## Voir aussi

- [Installer et configurer Fail2ban sur un serveur Ubuntu/Debian]({% post_url 2025-09-21-Installer-et-configurer-Fail2ban-sur-Ubuntu-Debian %})
- [Linux : Programmer une tâche avec cron]({% post_url 2025-10-11-Linux-programmer-une-tache-avec-cron %})
- [Automatic updates](https://ubuntu.com/server/docs/how-to/software/automatic-updates/), dans la documentation d'Ubuntu Server
- [UnattendedUpgrades](https://wiki.debian.org/UnattendedUpgrades), sur le wiki Debian
- [Le README d'unattended-upgrades](https://github.com/mvo5/unattended-upgrades), avec la liste des options
