---
layout: article
title: "Linux : Comment changer le hostname en ligne de commande (Ubuntu/Debian)"
description: "Changer le hostname sous Ubuntu ou Debian en ligne de commande avec hostnamectl ou sans systemd, mettre à jour /etc/hosts et choisir entre FQDN et nom court."
tags:
  - linux
  - ubuntu
  - debian
  - cli
  - sysadmin
  - réseau
author: Pierre Chopinet
---

Sur Debian ou Ubuntu, changer le nom d'une machine tient en une commande avec `hostnamectl`, sans redémarrage. Il reste ensuite à mettre à jour `/etc/hosts` et, sur une VM cloud, à s'assurer que cloud-init ne remettra pas l'ancien nom.
<!--more-->

Dans cet article :
- Afficher le nom actuel
- Changer le nom avec hostnamectl
- Mettre à jour /etc/hosts
- Sans hostnamectl
- Nom court ou FQDN
- Faut-il redémarrer ?
- Cloud-init et conteneurs

Pré-requis : un accès root ou sudo. Les exemples correspondent à Ubuntu 24.04 (systemd 255).

## Afficher le nom actuel

```bash
hostnamectl
```

Sans argument, `hostnamectl` affiche l'état de la machine (c'est l'équivalent de `hostnamectl status`). Sur une VM, la sortie contient notamment ces lignes :

```
 Static hostname: web01
       Icon name: computer-vm
  Virtualization: kvm
Operating System: Ubuntu 24.04.3 LTS
    Architecture: x86-64
```

Une ligne `Transient hostname` n'apparaît que si le nom courant du noyau diffère du nom statique, et `Pretty hostname` seulement si un nom d'affichage a été défini (on y revient plus bas).

Pour afficher simplement le nom, sans passer par systemd :

```bash
hostname          # nom courant, celui du noyau
uname -n          # le même nom, lu via uname
cat /etc/hostname # nom configuré, appliqué au démarrage
```

## Changer le nom avec hostnamectl

```bash
sudo hostnamectl set-hostname mon-serveur
```

La commande écrit le nouveau nom dans `/etc/hostname`, qui est lu à chaque démarrage, et l'applique tout de suite au noyau. Depuis systemd 249, la documentation présente cette commande sous la forme `hostnamectl hostname mon-serveur`. L'ancienne forme `set-hostname` est marquée obsolète dans le code mais toujours acceptée, et c'est la seule qui fonctionne sur les versions plus anciennes de systemd : autant la garder dans vos scripts.

Le nom doit être un nom DNS valide : lettres minuscules, chiffres et tirets, sans espace ni accent, 64 caractères au maximum.

On vérifie avec :

```bash
hostnamectl
hostname
```

Attention, un shell déjà ouvert garde l'ancien nom dans son invite : bash lit le nom de la machine une seule fois, à son lancement. Ouvrez un nouveau terminal ou une nouvelle session SSH (ou lancez `exec bash`) pour voir le nouveau nom.

systemd distingue en fait trois noms. Le nom statique est celui qu'on vient de modifier. Le nom « joli » (pretty) sert uniquement à l'affichage et peut contenir des espaces et des majuscules ; il est stocké dans `/etc/machine-info` :

```bash
sudo hostnamectl set-hostname "Mon Serveur de Paris" --pretty
```

Ce nom n'est jamais utilisé sur le réseau : pour les scripts, la résolution de noms ou SSH, c'est le nom statique qui compte. Enfin, le nom transitoire (transient) est celui que peut fournir le réseau : NetworkManager ou systemd-networkd savent récupérer un nom envoyé par le serveur DHCP. systemd ne s'en sert que si aucun nom statique n'est configuré. Sur un serveur dont le fichier `/etc/hostname` est rempli, c'est donc toujours le nom statique qui l'emporte.

## Mettre à jour /etc/hosts

`hostnamectl` ne touche pas à `/etc/hosts`. Or sur Debian et Ubuntu, le nom de la machine y est associé à l'adresse `127.0.1.1`. Si cette ligne contient encore l'ancien nom, le nouveau nom ne peut pas être résolu en local, ce qui gêne les programmes qui en ont besoin (`hostname -f` par exemple, voir plus bas).

```bash
sudo nano /etc/hosts
```

Le début du fichier doit ressembler à ceci :

```
127.0.0.1   localhost
127.0.1.1   mon-serveur
```

Ne modifiez pas la ligne `127.0.0.1 localhost` : le nom de la machine a sa propre ligne, en `127.0.1.1`.

## Sans hostnamectl

Sur un système sans systemd, on fait le travail à la main :

```bash
echo "mon-serveur" | sudo tee /etc/hostname
sudo hostname mon-serveur
```

La première commande rend le changement permanent, la seconde l'applique immédiatement : `hostname` ne modifie que le nom courant du noyau, qui sera perdu au prochain redémarrage si `/etc/hostname` n'a pas été mis à jour. Il faut ensuite corriger `/etc/hosts` comme vu plus haut.

## Nom court ou FQDN

Le nom court (`mon-serveur`) suffit dans la plupart des cas. Le nom complet, ou FQDN (`mon-serveur.example.com`), devient utile avec un DNS interne, Kerberos ou certains certificats.

On peut techniquement passer un FQDN à `hostnamectl`, mais la documentation de systemd recommande un nom sans point dans `/etc/hostname`. La page de manuel de `hostname` sur Debian conseille plutôt de définir le FQDN dans `/etc/hosts` (ou dans le DNS), en faisant du nom court un alias du nom complet. Le FQDN vient en premier sur la ligne, suivi du nom court :

```
127.0.1.1   mon-serveur.example.com mon-serveur
```

On vérifie avec `hostname -f` :

```bash
hostname -f
```

```
mon-serveur.example.com
```

`hostname -s` donne le nom court et `hostname -d` le domaine (`example.com`). Sans cette ligne dans `/etc/hosts` (ni entrée DNS), `hostname -f` échoue avec le message `hostname: Name or service not known`.

## Faut-il redémarrer ?

Non. `hostnamectl` applique le nouveau nom immédiatement. Par contre, certains services lisent le nom de la machine une seule fois, à leur démarrage, et continuent d'utiliser l'ancien : redémarrez-les, ou redémarrez la machine si vous ne savez pas lesquels sont concernés.

Côté SSH, il n'y a rien à faire. Les clés d'hôte du serveur ne dépendent pas de son nom, donc le changement ne déclenche pas l'avertissement `REMOTE HOST IDENTIFICATION HAS CHANGED!` sur les postes clients. Ce message n'apparaît que si la clé présentée ne correspond pas à celle enregistrée pour ce nom ou cette adresse (après une réinstallation, ou si le nom désignait auparavant une autre machine). Si vous vous connectez avec le nouveau nom, ssh vous demandera simplement de confirmer la clé, comme pour une machine qu'il ne connaît pas encore.

## Cloud-init et conteneurs

Sur une VM cloud, cloud-init met à jour `/etc/hostname` à chaque démarrage à partir des métadonnées de l'hébergeur. Sa documentation indique qu'il ne touche plus au fichier une fois qu'on l'a modifié à la main, mais pour qu'il laisse le nom tranquille à coup sûr, on active l'option `preserve_hostname` :

```bash
sudo sed -i 's/^preserve_hostname: false/preserve_hostname: true/' /etc/cloud/cloud.cfg
```

Autre piège sur ces machines : si `/etc/hosts` commence par un commentaire qui parle de `manage_etc_hosts`, cloud-init régénère le fichier à chaque démarrage. Il faut alors faire les modifications dans le modèle `/etc/cloud/templates/hosts.debian.tmpl`, ou désactiver `manage_etc_hosts`.

Dans un conteneur, enfin, le nom est fixé par le moteur au lancement : avec Docker, on le choisit avec l'option `--hostname` de `docker run`, plutôt que de le modifier depuis l'intérieur du conteneur.

## Voir aussi

- [Installer et configurer Fail2ban sur un serveur Ubuntu/Debian]({% post_url 2025-09-21-Installer-et-configurer-Fail2ban-sur-Ubuntu-Debian %})
- [Activer les mises à jour de sécurité automatiques sur Ubuntu/Debian]({% post_url 2025-12-19-Activer-les-mises-a-jour-de-securite-automatiques-sur-Ubuntu-Debian %})
- [systemd : hostnamectl](https://www.freedesktop.org/software/systemd/man/latest/hostnamectl.html)
- [systemd : hostname(5), le fichier /etc/hostname](https://www.freedesktop.org/software/systemd/man/latest/hostname.html)
- [Debian Wiki : Hostname](https://wiki.debian.org/Hostname)
- [cloud-init : modules Set Hostname, Update Hostname et Update Etc Hosts](https://docs.cloud-init.io/en/latest/reference/modules.html)
