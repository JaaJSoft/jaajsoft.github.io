---
layout: article
title: "Installer et configurer Fail2ban sur un serveur Ubuntu/Debian"
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
  - tutoriel
---

Fail2ban est un outil indispensable pour protéger un serveur exposé sur Internet contre les tentatives de brute‑force (SSH, HTTP, etc.).
<!--more-->
L'outil surveille les logs, détecte les échecs répétés, puis bannit temporairement l’adresse IP via le pare‑feu.

Objectifs de cet article :

- Installer Fail2ban sur Ubuntu/Debian
- Comprendre sa philosophie (filters, jails, actions)
- Activer une protection SSH simple et efficace
- Comment vérifier le bon fonctionnement, dépanner et des pistes pour aller plus loin (recidive, notifications)

Pré‑requis :

- Un serveur Debian/Ubuntu avec accès sudo
- Un pare‑feu actif (UFW recommandé sur Ubuntu) ou iptables/nftables

---

## 1) Installation et démarrage

Sur Debian/Ubuntu récents :

```bash
sudo apt update
sudo apt install fail2ban
```

Activer au démarrage et lancer :

```bash
sudo systemctl enable --now fail2ban
sudo systemctl status fail2ban
```

---

## 2) Philosophie et fichiers importants

- Filters (filtres) : regex qui détectent les lignes suspectes dans les logs (ex : `/etc/fail2ban/filter.d/sshd.conf`).
- Actions : que fait Fail2ban quand il bannit ? (ex : ajouter une règle firewall via UFW/iptables/nftables).
- Jails : association filtre + action + paramètres (bantime, findtime, maxretry, etc.).

Ne modifiez jamais les fichiers `.conf` fournis par le paquet ; créez des `.local` pour surcharger :

- Fichier global : `/etc/fail2ban/jail.local`
- Dossiers de configuration utilisateurs : `/etc/fail2ban/jail.d/*.local` et `*.conf` (priorité `.local`)

Les journaux observés peuvent être des fichiers (ex : `/var/log/auth.log`) ou le journal `systemd` (backend `systemd`).

---

## 3) Configuration de base: protéger SSH

Créer un fichier `/etc/fail2ban/jail.local` :

```ini
[DEFAULT]
# Temps de bannissement (ex : 10 minutes)
bantime = 10m
# Fenêtre d’observation des échecs
findtime = 10m
# Nombre d’échecs avant ban
maxretry = 5
# Adresse(s) à ne jamais bannir (mettez votre IP publique)
ignoreip = 127.0.0.1/8 ::1
# Lecture de logs via systemd (souvent plus fiable sur Ubuntu/Debian)
backend = systemd

# Choisissez l’action selon votre pare‑feu (voir plus bas)
# banaction = ufw
# banaction = iptables-multiport
# banaction = nftables-multiport

[sshd]
enabled = true
port    = 22
# Le filtre sshd est fourni par défaut
filter  = sshd
# Journal : laissez Fail2ban deviner avec backend=systemd
# (Sinon: logpath = /var/log/auth.log)
```

Sauvegardez, puis rechargez la configuration :

```bash
sudo systemctl reload fail2ban
# ou
sudo fail2ban-client reload
```

### Choisir l’action (UFW / iptables / nftables)

Vérifiez quel firewall est installé :

```bash
sudo ufw status               # si actif, privilégier banaction=ufw
sudo which iptables           # compat couche iptables-nft possible
sudo which nft                # présence de nftables
```

- Si UFW est votre pare‑feu :

```ini
# dans [DEFAULT]
banaction = ufw
```

- Sans UFW, sur systèmes modernes (nftables) :

```ini
banaction = nftables-multiport
```

- Anciennes configs encore basées iptables :

```ini
banaction = iptables-multiport
```

---

## 4) Vérifier que ça fonctionne

- État global :

```bash
sudo fail2ban-client status
```

- Détail du jail SSH :

```bash
sudo fail2ban-client status sshd
```

- Voir les IP bannies, les tentatives récentes, etc. Exemple de sortie :

```
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  `- Total failed: 12
`- Actions
   |- Currently banned: 2
   `- Total banned: 4
```

- Débannir manuellement une IP (ex : 203.0.113.10) :

```bash
sudo fail2ban-client set sshd unbanip 203.0.113.10
```

- Bannir manuellement pour tester :

```bash
sudo fail2ban-client set sshd banip 203.0.113.10
```

- Logs Fail2ban :

```bash
sudo journalctl -u fail2ban -e
# ou
sudo tail -f /var/log/fail2ban.log
```

---

## 5) Ajuster la sensibilité (bantime, findtime, maxretry)

- bantime : durée de ban (ex : 10m, 1h, 24h). Unité : s, m, h, d, w.
- findtime : fenêtre pendant laquelle on compte les échecs.
- maxretry : nombre d’échecs permis dans la fenêtre.

Exemple « plus strict » :

```ini
[DEFAULT]
bantime = 1h
findtime = 15m
maxretry = 3
```

Astuce : mettez des valeurs plus permissives au début pour éviter de vous bannir et ajustez progressivement.

---

## 6) Whitelist: ignorer vos IPs

Ajoutez votre IP publique (ou votre bureau/VPN) dans `ignoreip` :

```ini
ignoreip = 127.0.0.1/8 ::1 198.51.100.42 203.0.113.0/24
```

Rechargez ensuite Fail2ban.

---

## 7) Aller plus loin

### 7.1 Jail « recidive » (récidivistes)

Le jail `recidive` bannit plus longtemps les IP qui déclenchent plusieurs bans dans la journée.

```ini
[recidive]
enabled  = true
logpath  = /var/log/fail2ban.log
bantime  = 1w
findtime = 1d
maxretry = 5
```

> Note : ce jail opère au‑dessus des autres jails (il lit le log Fail2ban). Utile en production.

### 7.2 Protéger Nginx

Plusieurs filtres sont fournis (selon la distribution) : `nginx-http-auth`, `nginx-botsearch`, etc. Exemple d’un jail basique :

```ini
[nginx-botsearch]
enabled = true
port    = http,https
logpath = /var/log/nginx/access.log
maxretry = 10
findtime = 10m
bantime  = 1h
```

Vérifiez que le filtre existe sur votre système :

```bash
ls /etc/fail2ban/filter.d/ | grep nginx
```

Sinon, vous pouvez créer vos propres filtres et jails dans `*.local`. Testez vos regex avec :

```bash
sudo fail2ban-regex /var/log/nginx/access.log /etc/fail2ban/filter.d/nginx-botsearch.conf
```

### 7.3 Notifications email

Vous pouvez recevoir un email lors d’un ban, en utilisant une action prédéfinie (ex : `action_mw`, `action_mwl`). Il faut disposer d’un agent de mail (ex : `postfix`).

Exemple :

```ini
[DEFAULT]
destemail = admin@example.com
sender = fail2ban@example.com
action = %(action_mwl)s
```

---

## 8) Bonnes pratiques

- Toujours utiliser des fichiers `.local`, ne pas modifier les `.conf` d’origine.
- Recharger la configuration après modification : `sudo fail2ban-client reload`.
- Tester les filtres avec `fail2ban-regex` si un jail ne matche pas.
- Surveiller `journalctl -u fail2ban` pour les erreurs (permissions de logs, chemins).
- Ne mettez pas un gros `bantime` au début , combinez plutôt avec `recidive`.
- Avec UFW, assurez‑vous que les ports légitimes restent ouverts (allow) avant d’activer des jails.

---

## 9) Résumé

- Installation : `apt install fail2ban`, service actif.
- Configuration propre dans `/etc/fail2ban/jail.local` ; activer `[sshd]`.
- Choisir l’action adaptée (UFW, nftables, iptables).
- Vérifier avec `fail2ban-client status` et les journaux.
- Aller plus loin : `recidive`, jails pour Nginx, notifications email.

---

Pour aller plus loin :

- [Documentation officielle Fail2ban](https://www.fail2ban.org/)
- [Debian Wiki – Fail2ban](https://wiki.debian.org/fail2ban)
- [Ubuntu Community Help – Fail2ban](https://help.ubuntu.com/community/Fail2ban)
- [Linux : Comment changer le hostname en ligne de commande (Ubuntu/Debian)]({% post_url 2025-09-13-Comment-changer-le-hostname-en-ligne-de-commande-sur-Ubuntu-ou-Debian %})
- [jq : Comment utiliser jq en ligne de commande]({% post_url 2025-09-17-Comment-utiliser-jq %})
- [K8s : Comment déployer un cluster kubernetes bare-metal]({% post_url 2021-01-27-Comment-déployer-Kubernetes %})




