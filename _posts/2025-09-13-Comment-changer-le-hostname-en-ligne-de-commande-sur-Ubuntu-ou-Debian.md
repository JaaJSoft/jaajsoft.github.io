---
layout: article
title: "Linux : Comment changer le hostname en ligne de commande (Ubuntu/Debian)"
author: Pierre Chopinet
tags:
  - linux
  - ubuntu
  - debian
  - cli
  - sysadmin
  - réseau
  - tutoriel
---

Comment changer rapidement le nom d’hôte (hostname) sous Ubuntu ou Debian, sans casse ni redémarrage inutile ? Ce guide vous montrera la méthode recommandée avec hostnamectl, les alternatives sans systemd, ainsi que les bonnes pratiques (mise à jour de /etc/hosts, FQDN, services).
<!--more-->

> Note : Ce guide a été testé sur Ubuntu 24.04 LTS et reste valable pour la plupart des versions récentes d’Ubuntu/Debian.

---

## 1) Vérifier le hostname actuel

```bash
hostnamectl status
```

Sortie typique (extrait) :

```
     Static hostname: web01
  Transient hostname: web01
           Icon name: computer-vm
             Chassis: vm
          Machine ID: ...
             Boot ID: ...
    Operating System: Ubuntu 24.04.3 LTS
              Kernel: Linux 6.8.0-...
        Architecture: x86-64
```

Commandes alternatives :

```bash
hostname       # affiche le nom d’hôte actuel
uname -n       # affiche le nodename du noyau
cat /etc/hostname
```

> Note : `hostnamectl` est fourni par systemd. Il est disponible sur Ubuntu/Debian modernes. Si vous n’avez pas `hostnamectl`, voir la section "Sans systemd" plus bas.

---

## 2) Changer le hostname avec hostnamectl (méthode recommandée)

Pour définir un nom d’hôte persistant ("static hostname") :

```bash
sudo hostnamectl set-hostname mon-serveur
```

Vérifiez :

```bash
hostnamectl status
hostname
```

> Note : Le changement est immédiat pour les nouveaux shells. Un shell déjà ouvert peut garder l’ancien prompt jusqu’à l’ouverture d’une nouvelle session (ou rechargement de l’invite).

### 2.1) Définir un "pretty hostname"

Le pretty hostname permet un nom d’affichage avec espaces/majuscule :

```bash
sudo hostnamectl set-hostname "Mon Serveur de Paris" --pretty
```

> Note : Le "pretty" n’est pas utilisé par la résolution DNS. Pour les usages réseau/scripts, utilisez le hostname statique sans espaces.

### 2.2) Définir un transient hostname

Le transient hostname est fourni par DHCP/NM et n’est pas persistant. Pour le définir manuellement (rarement nécessaire) :

```bash
sudo hostnamectl set-hostname mon-serveur-temp --transient
```

> Note : Le transient est réinitialisé au redémarrage ou par le client réseau. Préférez le "static" pour un nom stable.

---

## 3) Mettre à jour /etc/hosts (recommandé)

Il est souvent utile d’ajouter une entrée pour que le hostname se résolve en local, notamment pour certains services qui s’y attendent.

Éditez `/etc/hosts` :

```bash
sudo nano /etc/hosts
```

Assurez-vous d’avoir une ligne ressemblant à :

```
127.0.1.1   mon-serveur
```

Sur certaines distributions, on préfère mettre le FQDN puis l’alias court :

```
127.0.1.1   mon-serveur.exemple.local mon-serveur
```

> Note : Ne remplacez pas la ligne `127.0.0.1 localhost`. Ajoutez une entrée séparée pour votre hostname (souvent `127.0.1.1` sur Debian/Ubuntu).

---

## 4) Méthode classique (sans hostnamectl)

Si vous n’avez pas systemd ou préférez la méthode "fichiers" :

1) Écrire le nouveau nom dans `/etc/hostname` :

```bash
echo "mon-serveur" | sudo tee /etc/hostname
```

2) Mettre à jour `/etc/hosts` comme vu plus haut.

3) Appliquer le changement immédiatement (sans reboot) :

```bash
sudo hostname mon-serveur
```

> Note : La commande `hostname` modifie le nom courant jusqu’au prochain reboot. Le fichier `/etc/hostname` garantit la persistance au redémarrage.

---

## 5) FQDN ou short name ?

- Short name : `mon-serveur` (simple, recommandé en local).
- FQDN : `mon-serveur.exemple.local` (utile si vous avez un DNS interne, Kerberos, certificats, etc.).

Avec `hostnamectl`, on peut définir un FQDN directement en static hostname :

```bash
sudo hostnamectl set-hostname mon-serveur.exemple.local
```

---

## 6) Faut-il redémarrer ?

La plupart du temps, non. `hostnamectl` applique immédiatement le nouveau nom. Certains services lisent le hostname au démarrage ; au besoin, redémarrez le service concerné ou le bus hostnamed :

```bash
sudo systemctl restart systemd-hostnamed
```

> Note : Des sessions SSH en cours peuvent continuer d’afficher l’ancien hostname dans le prompt. Ouvrez une nouvelle session pour voir le changement.

---

## 7) Points d'attention

- Cloud-init (VM/cloud) : certains environnements réécrivent le hostname au boot.
  - Désactiver la gestion du hostname par cloud-init si vous voulez le contrôler :
    ```bash
    sudo sed -i 's/^preserve_hostname: false/preserve_hostname: true/' /etc/cloud/cloud.cfg
    ```
- NetworkManager/DHCP : un serveur DHCP peut imposer un transient hostname.
- Conteneurs (Docker/LXC) : le hostname vient souvent du runtime. Dans un conteneur Docker, utilisez `--hostname` au `docker run`.
- SSH known_hosts : si vous changez le hostname d’un serveur SSH que vous contactez par nom (et/ou son IP), vous pourriez avoir un avertissement `REMOTE HOST IDENTIFICATION HAS CHANGED!`. Mettez à jour `~/.ssh/known_hosts`.
- Droits sudo : la plupart des commandes nécessitent `sudo`.

---

## Récapitulatif

- Méthode recommandée : `sudo hostnamectl set-hostname <nouveau_nom>` puis modifier `/etc/hosts`.
- Vérifier : `hostnamectl status`, `hostname`.
- Pas de reboot nécessaire, sauf cas particuliers.
- Attention à cloud-init, DHCP, conteneurs et FQDN.


---

## Pour aller plus loin

- [Ubuntu (manpage) - hostnamectl (24.04 LTS « noble »)](https://manpages.ubuntu.com/manpages/noble/en/man1/hostnamectl.1.html)
- [Debian Wiki - Hostname](https://wiki.debian.org/Hostname)
- [Debian Wiki - Changer le hostname](https://wiki.debian.org/HowTo/ChangeHostname)
- [systemd - Documentation hostnamectl](https://www.freedesktop.org/software/systemd/man/latest/hostnamectl.html)
- [systemd - hostnamed.service (bus D‑Bus du hostname)](https://www.freedesktop.org/software/systemd/man/latest/hostnamed.service.html)
- [cloud-init - preserve_hostname et gestion du hostname](https://cloudinit.readthedocs.io/en/latest/reference/config.html#preserve-hostname)
- [Docker - option --hostname pour les conteneurs](https://docs.docker.com/reference/cli/docker/container/run/#hostname)


---

## Pour aller plus loin sur linux

- [Installer et configurer Fail2ban sur un serveur Ubuntu/Debian]({% post_url 2025-09-21-Installer-et-configurer-Fail2ban-sur-Ubuntu-Debian %})
- [Activer les mises à jour de sécurité automatiques sur Ubuntu/Debian]({% post_url 2025-12-19-Activer-les-mises-a-jour-de-securite-automatiques-sur-Ubuntu-Debian %})
