---
layout: article
title: "Installer et configurer Fail2ban sur un serveur Ubuntu/Debian"
description: "Installer et configurer Fail2ban sur Ubuntu ou Debian pour protéger SSH contre le brute-force : jails, bantime, findtime, maxretry et whitelist."
author: Pierre Chopinet
tags:
  - linux
  - fail2ban
  - sécurité
  - ssh
  - firewall
  - debian
  - ubuntu
  - serveur
---

Un serveur exposé sur Internet reçoit en permanence des tentatives de connexion SSH par force brute. Fail2ban surveille les journaux, repère les échecs répétés et bannit temporairement les adresses IP concernées via le pare-feu.
<!--more-->

On va l'installer sur Ubuntu ou Debian, protéger SSH, puis ajouter le jail `recidive`, un jail pour Nginx et les notifications par mail.

Dans cet article :
- Installation
- Filtres, actions et jails
- Protéger SSH
- Choisir l'action de bannissement
- Vérifier que ça fonctionne
- Ajuster bantime, findtime et maxretry
- Ne pas bannir ses propres adresses
- Le jail recidive
- Protéger Nginx
- Recevoir un mail à chaque bannissement

Pré-requis : un serveur Debian ou Ubuntu avec un accès sudo. Les configurations ont été vérifiées avec Fail2ban 1.0.2, la version fournie par Ubuntu 24.04.

## Installation

```bash
sudo apt update
sudo apt install fail2ban
```

Le paquet active et démarre le service. Pour s'en assurer et voir son état :

```bash
sudo systemctl enable --now fail2ban
sudo systemctl status fail2ban
```

Sur Ubuntu 24.04, la protection SSH est même active dès l'installation, grâce au fichier `/etc/fail2ban/jail.d/defaults-debian.conf` livré par le paquet :

```ini
[DEFAULT]
banaction = nftables
banaction_allports = nftables[type=allports]
backend = systemd

[sshd]
enabled = true
```

Les lignes `banaction` et `backend` n'existent dans ce fichier que depuis la version 1.0.2-3 du paquet Debian. Le paquet de Debian 12, antérieur à cette version, ne les contient pas : c'est une bonne raison de les écrire explicitement dans votre propre configuration.

## Filtres, actions et jails

Fail2ban repose sur trois notions. Un filtre est un ensemble d'expressions régulières qui reconnaissent les lignes d'échec dans un journal : `/etc/fail2ban/filter.d/sshd.conf`, par exemple, repère les connexions SSH ratées. Une action, rangée dans `/etc/fail2ban/action.d/`, décrit comment bannir et débannir une adresse : ajouter une règle nftables, iptables ou UFW, envoyer un mail... Enfin, un jail associe un filtre, une ou plusieurs actions et des paramètres comme la durée du bannissement ou le nombre d'échecs qui le déclenche.

Les fichiers `.conf` appartiennent au paquet et peuvent être remplacés lors d'une mise à jour : on ne les modifie pas. On écrit ses réglages dans des fichiers `.local`, qui ne contiennent que ce qu'on veut changer. Pour les jails, les fichiers sont lus dans cet ordre, chacun l'emportant sur les précédents :

```
jail.conf
jail.d/*.conf
jail.local
jail.d/*.local
```

Un `jail.local` passe donc après le `defaults-debian.conf` vu plus haut.

## Protéger SSH

Créez le fichier `/etc/fail2ban/jail.local` :

```ini
[DEFAULT]
# Temps de bannissement (ex : 10 minutes)
bantime = 10m
# Fenêtre d'observation des échecs
findtime = 10m
# Nombre d'échecs avant ban
maxretry = 5
# Adresse(s) à ne jamais bannir (mettez votre IP publique)
ignoreip = 127.0.0.1/8 ::1
# Lecture de logs via systemd (souvent plus fiable sur Ubuntu/Debian)
backend = systemd

# Choisissez l'action selon votre pare-feu (voir plus bas)
# banaction = ufw
# banaction = iptables-multiport
# banaction = nftables-multiport

[sshd]
enabled = true
port    = 22
# Le filtre sshd est fourni par défaut
filter  = sshd
# Journal : laissez Fail2ban deviner avec backend=systemd
# (Sinon : logpath = /var/log/auth.log)
```

Avec ces valeurs, qui sont celles par défaut de `jail.conf`, une adresse qui accumule 5 échecs en 10 minutes est bannie pendant 10 minutes. Si votre serveur SSH n'écoute pas sur le port 22, indiquez le bon port dans `port` : les actions nftables et iptables ne bloquent que les ports du jail.

La ligne `backend = systemd` fait lire les échecs dans le journal de systemd plutôt que dans un fichier comme `/var/log/auth.log`, qui n'existe que si rsyslog est installé. Ce backend a besoin du module Python de systemd. C'est une dépendance du paquet depuis la version 1.0.2-3 ; avec un paquet plus ancien comme celui de Debian 12, installez-le au besoin avec `sudo apt install python3-systemd`. Sans ce module, le jail ne démarre pas et Fail2ban signale `Failed to initialize any backend for Jail 'sshd'`.

On vérifie la configuration, puis on la recharge :

```bash
sudo fail2ban-client -t
sudo systemctl reload fail2ban
```

`fail2ban-client -t` teste la configuration sans rien appliquer et affiche `OK: configuration test is successful` si tout va bien. `systemctl reload fail2ban` se contente d'appeler `fail2ban-client reload` : les deux commandes sont équivalentes.

## Choisir l'action de bannissement

L'action utilisée par défaut dépend du paquet : `nftables` sur Ubuntu 24.04 (le `defaults-debian.conf` vu plus haut), `iptables-multiport` dans le `jail.conf` d'origine. Pour savoir ce qui est installé sur la machine :

```bash
sudo ufw status               # si actif, privilégier banaction=ufw
which iptables                # compat couche iptables-nft possible
which nft                     # présence de nftables
```

Si UFW gère le pare-feu de la machine, utilisez son action : les bannissements apparaîtront alors dans `sudo ufw status`.

```ini
# dans [DEFAULT]
banaction = ufw
```

Sinon, `nftables-multiport` (identique à l'action `nftables` utilisée par Ubuntu) convient aux systèmes récents, et `iptables-multiport` aux configurations encore basées sur iptables. Fail2ban crée alors ses propres règles, sans toucher aux vôtres.

## Vérifier que ça fonctionne

L'état global liste les jails actifs :

```bash
sudo fail2ban-client status
```

```
Status
|- Number of jail:      1
`- Jail list:   sshd
```

Pour tester, on peut bannir une adresse à la main, puis regarder l'état du jail :

```bash
sudo fail2ban-client set sshd banip 203.0.113.10
sudo fail2ban-client status sshd
```

```
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     0
|  `- Journal matches:  _SYSTEMD_UNIT=sshd.service + _COMM=sshd
`- Actions
   |- Currently banned: 1
   |- Total banned:     1
   `- Banned IP list:   203.0.113.10
```

La ligne `Journal matches` confirme que le jail lit le journal de systemd ; avec un backend fichier, on verrait à la place une ligne `File list` avec le chemin du journal. Pour débannir l'adresse :

```bash
sudo fail2ban-client set sshd unbanip 203.0.113.10
```

La commande `sudo fail2ban-client unban 203.0.113.10` fait la même chose dans tous les jails à la fois.

Le compteur `Total failed` doit augmenter quand une connexion SSH échoue. Faites un essai avec un mauvais mot de passe depuis une machine dont l'adresse n'est pas dans `ignoreip`, moins de 5 fois pour ne pas vous bannir. Si le compteur reste à 0, le jail ne voit pas les échecs : le problème vient presque toujours du backend ou du chemin du journal.

Fail2ban écrit son propre journal dans `/var/log/fail2ban.log` ; les erreurs de démarrage du service sont aussi visibles avec `journalctl` :

```bash
sudo journalctl -u fail2ban -e
# ou
sudo tail -f /var/log/fail2ban.log
```

## Ajuster bantime, findtime et maxretry

Trois paramètres règlent la sensibilité d'un jail :

- `bantime` : durée du bannissement ;
- `findtime` : fenêtre pendant laquelle on compte les échecs ;
- `maxretry` : nombre d'échecs qui, dans cette fenêtre, déclenche le bannissement.

Les durées s'écrivent en secondes ou avec une unité : `s`, `m`, `h`, `d` ou `w` pour les semaines. Attention, `m` signifie minutes : les mois s'écrivent `mo`. En cas de doute, `fail2ban-client --str2sec 1w` affiche la conversion en secondes (604800).

Exemple plus strict :

```ini
[DEFAULT]
bantime = 1h
findtime = 15m
maxretry = 3
```

Commencez avec des valeurs souples pour éviter de vous bannir vous-même, puis resserrez progressivement. Plutôt qu'un `bantime` très long dès le départ, vous pouvez aussi allonger la durée pour les adresses qui reviennent, avec le jail `recidive` présenté plus bas ou avec l'option `bantime.increment = true`. Cette option, disponible depuis Fail2ban 0.11 et décrite dans les commentaires de `jail.conf`, augmente la durée à chaque nouveau bannissement d'une même adresse : par défaut, elle double.

## Ne pas bannir ses propres adresses

Ajoutez votre IP publique (ou celle de votre bureau, de votre VPN) à la liste `ignoreip` de la section `[DEFAULT]` :

```ini
ignoreip = 127.0.0.1/8 ::1 198.51.100.42 203.0.113.0/24
```

La liste accepte des adresses, des plages CIDR et des noms DNS, séparés par des espaces. Les adresses de la machine elle-même sont déjà ignorées par défaut (option `ignoreself`). Rechargez ensuite Fail2ban.

## Le jail recidive

Le jail `recidive` lit le journal de Fail2ban lui-même et bannit plus longtemps, sur tous les ports, les adresses qui se font bannir à répétition par les autres jails :

```ini
[recidive]
enabled  = true
# Nécessaire si backend=systemd est défini dans [DEFAULT] :
# le backend systemd ignore logpath, on force la lecture du fichier
backend  = auto
logpath  = /var/log/fail2ban.log
bantime  = 1w
findtime = 1d
maxretry = 5
```

Ici, une adresse bannie 5 fois dans la journée l'est ensuite pour une semaine. La ligne `backend = auto` est importante : avec le `backend = systemd` hérité de `[DEFAULT]`, Fail2ban ignorerait `logpath` et chercherait les bannissements dans le journal de systemd, alors qu'il les écrit dans `/var/log/fail2ban.log`. Le jail démarrerait sans erreur, mais ne bannirait jamais personne.

## Protéger Nginx

Fail2ban 1.0.2 fournit quatre filtres pour Nginx, que l'on peut lister :

```bash
ls /etc/fail2ban/filter.d/ | grep nginx
```

Le filtre `nginx-botsearch` repère les robots qui cherchent des pages connues (`wp-login.php`, phpMyAdmin, `cgi-bin`...) et tombent sur une erreur 404. Dans `jail.conf`, le jail du même nom lit les journaux d'erreurs de Nginx, mais le filtre reconnaît aussi le format de `access.log`, utilisé ici :

```ini
[nginx-botsearch]
enabled  = true
# Nginx écrit dans des fichiers : pas de backend systemd ici
backend  = auto
port     = http,https
logpath  = /var/log/nginx/access.log
maxretry = 10
findtime = 10m
bantime  = 1h
```

Comme pour `recidive`, la ligne `backend = auto` est nécessaire quand `backend = systemd` est défini dans `[DEFAULT]` : sans elle, le jail surveille le journal de systemd et ne voit jamais les requêtes enregistrées dans `access.log`.

Avant d'activer un jail, et surtout pour mettre au point vos propres filtres (dans des fichiers `.local`), testez le filtre sur un vrai journal avec `fail2ban-regex` :

```bash
sudo fail2ban-regex /var/log/nginx/access.log /etc/fail2ban/filter.d/nginx-botsearch.conf
```

La commande indique combien de lignes ont été reconnues (`Lines: ... matched, ... missed`) et affiche une partie des lignes non reconnues (toutes avec `--print-all-missed`). Pour un jail qui lit le journal de systemd, on remplace le fichier par `systemd-journal` : `sudo fail2ban-regex systemd-journal sshd`.

## Recevoir un mail à chaque bannissement

Fail2ban peut envoyer un mail à chaque bannissement avec les actions prédéfinies `action_mw` et `action_mwl` :

```ini
[DEFAULT]
destemail = admin@example.com
sender = fail2ban@example.com
action = %(action_mwl)s
```

Les deux actions bannissent l'adresse comme d'habitude et envoient un mail contenant le résultat de `whois` pour cette adresse ; `action_mwl` y ajoute les lignes du journal qui la concernent. L'envoi passe par la commande `sendmail` : il faut un serveur de mail sur la machine (Postfix par exemple) et la commande `whois`.

Attention, `action_mwl` cherche ces lignes dans le fichier indiqué par `logpath`. Avec le backend `systemd` et sans rsyslog, `/var/log/auth.log` n'existe pas et cette partie du mail reste vide : `action_mw` suffit alors.

## Voir aussi

- [Activer les mises à jour de sécurité automatiques sur Ubuntu/Debian]({% post_url 2025-12-19-Activer-les-mises-a-jour-de-securite-automatiques-sur-Ubuntu-Debian %})
- [Linux : Comment changer le hostname en ligne de commande (Ubuntu/Debian)]({% post_url 2025-09-13-Comment-changer-le-hostname-en-ligne-de-commande-sur-Ubuntu-ou-Debian %})
- [Fail2ban sur GitHub (code source et wiki)](https://github.com/fail2ban/fail2ban)
- [Page de manuel jail.conf(5) (Ubuntu 24.04)](https://manpages.ubuntu.com/manpages/noble/en/man5/jail.conf.5.html)
