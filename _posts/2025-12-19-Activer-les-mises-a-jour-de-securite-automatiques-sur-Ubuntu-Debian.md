---
layout: article
title: "Activer les mises à jour de sécurité automatiques sur Ubuntu/Debian"
author: Pierre Chopinet
tags:
  - linux
  - ubuntu
  - debian
  - sécurité
  - administration
---

Maintenir un système Linux à jour est crucial pour la sécurité, mais gérer manuellement les mises à jour peut être fastidieux. Ubuntu et Debian offrent des mécanismes pour automatiser les mises à jour de sécurité, garantissant que les correctifs critiques sont appliqués sans intervention humaine. Ce guide vous montre comment configurer et gérer ces mises à jour automatiques en toute sécurité.
<!--more-->

Dans ce guide, vous allez apprendre à :

- Comprendre les différents types de mises à jour (sécurité, recommandées, toutes)
- Installer et configurer `unattended-upgrades`
- Personnaliser le comportement des mises à jour automatiques
- Configurer les notifications par email
- Gérer les redémarrages automatiques si nécessaire
- Vérifier et surveiller les mises à jour appliquées
- Résoudre les problèmes courants

Prérequis :
- Ubuntu 16.04+ ou Debian 9+ (la plupart des distributions récentes)
- Accès root ou sudo

---

## Pourquoi activer les mises à jour automatiques ?

**Avantages**

- **Sécurité renforcée** : Les failles de sécurité sont corrigées rapidement, réduisant la fenêtre d'exposition aux attaques.
- **Gain de temps** : Plus besoin de surveiller et d'appliquer manuellement les mises à jour de sécurité.
- **Conformité** : Facilite le respect des politiques de sécurité et de conformité (ISO 27001, PCI-DSS, etc.).
- **Stabilité** : Les mises à jour de sécurité sont généralement testées et ne cassent pas le système.

**Précautions**

- **Mises à jour de paquets critiques** : Certaines mises à jour peuvent nécessiter un redémarrage (kernel, libc, systemd).
- **Applications personnalisées** : Les mises à jour peuvent potentiellement affecter des configurations spécifiques.
- **Bande passante** : Les téléchargements automatiques consomment de la bande passante.

**Recommandation** : Activez les mises à jour automatiques de sécurité uniquement (pas toutes les mises à jour) pour minimiser les risques.

---

## 1) Vérifier l'état actuel

Avant de commencer, vérifiez si `unattended-upgrades` est déjà installé :

```bash
# Vérifier si le paquet est installé
dpkg -l | grep unattended-upgrades

# Vérifier le statut du service
systemctl status unattended-upgrades
```

Si le paquet n'est pas installé, vous verrez un message vide ou une erreur.

---

## 2) Installation de unattended-upgrades

**Sur Ubuntu**

Sur Ubuntu, `unattended-upgrades` est généralement préinstallé. Si ce n'est pas le cas :

```bash
sudo apt update
sudo apt install unattended-upgrades
```

**Sur Debian**

```bash
sudo apt update
sudo apt install unattended-upgrades apt-listchanges
```

**Note** : `apt-listchanges` est optionnel mais recommandé pour voir les changements avant qu'ils ne soient appliqués.

---

## 3) Configuration de base (mises à jour de sécurité uniquement)

**Méthode rapide (recommandée pour débuter)**

Utilisez l'outil interactif de configuration :

```bash
sudo dpkg-reconfigure -plow unattended-upgrades
```

Sélectionnez **Yes** (Oui) pour activer les mises à jour automatiques de sécurité.

Cela crée automatiquement le fichier `/etc/apt/apt.conf.d/20auto-upgrades` avec le contenu suivant :

```
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

**Vérification**

```bash
cat /etc/apt/apt.conf.d/20auto-upgrades
```

Vous devriez voir :
- `Update-Package-Lists "1"` : Met à jour la liste des paquets quotidiennement
- `Unattended-Upgrade "1"` : Active les mises à jour automatiques

---

## 4) Configuration avancée

Le fichier principal de configuration est `/etc/apt/apt.conf.d/50unattended-upgrades`.

Ouvrez-le pour personnaliser les options :

```bash
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

**a) Choisir les sources de mise à jour**

Par défaut, seules les mises à jour de sécurité sont activées. Voici les lignes importantes (décommentez selon vos besoins) :

***Ubuntu***

```conf
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}";           // Mises à jour normales (déconseillé)
    "${distro_id}:${distro_codename}-security";  // Mises à jour de sécurité (recommandé)
    "${distro_id}ESMApps:${distro_codename}-apps-security"; // ESM (Ubuntu Pro)
    "${distro_id}ESM:${distro_codename}-infra-security";     // ESM (Ubuntu Pro)
//  "${distro_id}:${distro_codename}-updates";   // Mises à jour recommandées
//  "${distro_id}:${distro_codename}-proposed";  // Paquets en test (NE PAS ACTIVER)
//  "${distro_id}:${distro_codename}-backports"; // Backports (NE PAS ACTIVER)
};
```

***Debian***

```conf
Unattended-Upgrade::Origins-Pattern {
    "origin=Debian,codename=${distro_codename},label=Debian";
    "origin=Debian,codename=${distro_codename},label=Debian-Security"; // Sécurité
    "origin=Debian,codename=${distro_codename}-security,label=Debian-Security"; // Ancien format
//  "origin=Debian,codename=${distro_codename}-updates"; // Mises à jour recommandées
};
```

**Recommandation** : Laissez uniquement les lignes avec `-security` activées (sans `//` devant).

**b) Mettre automatiquement à jour tous les paquets (optionnel)**

Si vous souhaitez aussi appliquer les mises à jour non-sécurité (déconseillé en production), décommentez la ligne :

```conf
// "${distro_id}:${distro_codename}-updates";
```

Enlevez le `//` :

```conf
"${distro_id}:${distro_codename}-updates";
```

**c) Exclure certains paquets (blacklist)**

Si vous voulez éviter la mise à jour automatique de certains paquets (ex: kernel, bases de données) :

```conf
Unattended-Upgrade::Package-Blacklist {
    "linux-image-*";    // Noyau Linux (nécessite un redémarrage)
    "mysql-server*";    // Serveur MySQL
    "postgresql*";      // PostgreSQL
    "nginx";            // Nginx
};
```

**Note** : Les wildcards (`*`) sont supportés.

**d) Redémarrage automatique (avec précaution)**

Certaines mises à jour (kernel, libc, systemd) nécessitent un redémarrage. Par défaut, le système ne redémarre pas automatiquement.

Pour activer le redémarrage automatique :

```conf
Unattended-Upgrade::Automatic-Reboot "true";
```

Pour limiter le redémarrage à une plage horaire (ex: 3h du matin) :

```conf
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "03:00";
```

**Recommandation** : Activez cela uniquement si vous gérez des serveurs non-critiques ou si vous avez une redondance. Pour les serveurs en production, préférez un redémarrage planifié manuel.

**e) Supprimer les paquets obsolètes et dépendances inutilisées**

```conf
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Remove-New-Unused-Dependencies "true";
```

Cela équivaut à exécuter `apt autoremove` après chaque mise à jour.

**f) Notifications par email**

Pour recevoir un email après chaque mise à jour :

```conf
Unattended-Upgrade::Mail "admin@example.com";
Unattended-Upgrade::MailReport "on-change"; // Options: always, only-on-error, on-change
```

**Prérequis** : Installez un agent de messagerie (ex: `postfix`, `sendmail`, `msmtp`) :

```bash
sudo apt install mailutils postfix
```

Lors de l'installation de `postfix`, choisissez **Internet Site** et entrez votre nom de domaine.

Testez l'envoi d'email :

```bash
echo "Test" | mail -s "Test email" admin@example.com
```

---

## 5) Configuration de la fréquence des mises à jour

Le fichier `/etc/apt/apt.conf.d/20auto-upgrades` contrôle la périodicité.

Ouvrez-le :

```bash
sudo nano /etc/apt/apt.conf.d/20auto-upgrades
```

Exemple de configuration :

```conf
APT::Periodic::Update-Package-Lists "1";       // Mise à jour de la liste des paquets (tous les 1 jour)
APT::Periodic::Download-Upgradeable-Packages "1"; // Télécharger les paquets en avance
APT::Periodic::Unattended-Upgrade "1";         // Appliquer les mises à jour (tous les 1 jour)
APT::Periodic::AutocleanInterval "7";          // Nettoyer les paquets obsolètes (tous les 7 jours)
```

Options disponibles :
- `"0"` : Désactivé
- `"1"` : Tous les jours
- `"7"` : Tous les 7 jours

**Recommandation** : Laissez `"1"` pour les mises à jour de sécurité (quotidiennes).

---

## 6) Activer et démarrer le service

**Méthode 1 : Via systemd (Ubuntu 18.04+, Debian 10+)**

```bash
# Activer le service au démarrage
sudo systemctl enable unattended-upgrades

# Démarrer le service
sudo systemctl start unattended-upgrades

# Vérifier le statut
sudo systemctl status unattended-upgrades
```

Sortie attendue :

```
● unattended-upgrades.service - Unattended Upgrades Shutdown
   Loaded: loaded (/lib/systemd/system/unattended-upgrades.service; enabled)
   Active: active (running)
```

**Méthode 2 : Via cron (anciennes versions)**

Les anciennes versions utilisent `apt-daily` et `apt-daily-upgrade` via cron :

```bash
# Vérifier les tâches cron
ls -l /etc/cron.daily/apt-compat
```

Pas besoin de configuration supplémentaire si les fichiers existent.

---

## 7) Tester manuellement les mises à jour automatiques

Vous pouvez simuler une exécution pour vérifier la configuration :

**Test à blanc (dry-run)**

```bash
sudo unattended-upgrade --dry-run --debug
```

Cela affiche les paquets qui seraient mis à jour sans les installer.

**Exécution réelle**

```bash
sudo unattended-upgrade --debug
```

Sortie attendue (exemple) :

```
Initial blacklisted packages:
Initial whitelisted packages:
Starting unattended upgrades script
Allowed origins are: ['o=Ubuntu,a=jammy-security', ...]
Packages that will be upgraded: libssl3 openssl
...
```

---

## 8) Vérifier les logs des mises à jour

Les logs sont disponibles dans `/var/log/unattended-upgrades/`.

**Voir les dernières mises à jour appliquées**

```bash
sudo cat /var/log/unattended-upgrades/unattended-upgrades.log
```

Exemple de sortie :

```
2025-12-19 10:15:32,123 INFO Starting unattended upgrades script
2025-12-19 10:15:35,456 INFO Packages that will be upgraded: libssl3 openssl
2025-12-19 10:16:12,789 INFO All upgrades installed
```

**Voir les erreurs**

```bash
sudo cat /var/log/unattended-upgrades/unattended-upgrades-dpkg.log
```

**Voir l'historique APT complet**

```bash
sudo cat /var/log/apt/history.log
```

---

## 9) Surveiller les redémarrages nécessaires

Après certaines mises à jour (kernel, libc, systemd), un redémarrage est nécessaire.

**Vérifier si un redémarrage est requis**

```bash
cat /var/run/reboot-required
```

Si le fichier existe, un redémarrage est nécessaire. Son contenu affiche :

```
*** System restart required ***
```

**Voir quels paquets nécessitent un redémarrage**

```bash
cat /var/run/reboot-required.pkgs
```

Exemple de sortie :

```
linux-image-5.15.0-58-generic
libc6
```

**Redémarrer manuellement**

```bash
sudo reboot
```

---

## 10) Désactiver les mises à jour automatiques

Si vous souhaitez désactiver temporairement ou définitivement les mises à jour automatiques :

**Méthode 1 : Via dpkg-reconfigure**

```bash
sudo dpkg-reconfigure -plow unattended-upgrades
```

Sélectionnez **No** (Non).

**Méthode 2 : Modifier le fichier de configuration**

```bash
sudo nano /etc/apt/apt.conf.d/20auto-upgrades
```

Changez les valeurs à `"0"` :

```conf
APT::Periodic::Update-Package-Lists "0";
APT::Periodic::Unattended-Upgrade "0";
```

**Méthode 3 : Désactiver le service**

```bash
sudo systemctl stop unattended-upgrades
sudo systemctl disable unattended-upgrades
```

---

## 11) Configuration complète recommandée

Voici une configuration équilibrée pour un serveur de production :

**`/etc/apt/apt.conf.d/20auto-upgrades`**

```conf
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Download-Upgradeable-Packages "1";
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::AutocleanInterval "7";
```

**`/etc/apt/apt.conf.d/50unattended-upgrades`**

```conf
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
};

Unattended-Upgrade::Package-Blacklist {
    "linux-image-*";
    "mysql-server*";
    "postgresql*";
    "nginx";
};

Unattended-Upgrade::AutoFixInterruptedDpkg "true";
Unattended-Upgrade::MinimalSteps "true";
Unattended-Upgrade::InstallOnShutdown "false";
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";

Unattended-Upgrade::Automatic-Reboot "false"; // Redémarrage manuel
Unattended-Upgrade::Automatic-Reboot-Time "03:00"; // Si activé : 3h du matin

Unattended-Upgrade::Mail "admin@example.com";
Unattended-Upgrade::MailReport "on-change";

Unattended-Upgrade::SyslogEnable "true";
Unattended-Upgrade::SyslogFacility "daemon";
```

---

## Cheatsheet

**Installation**

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades  # Config interactive
```

**Vérification**

```bash
sudo systemctl status unattended-upgrades
sudo unattended-upgrade --dry-run --debug
cat /var/log/unattended-upgrades/unattended-upgrades.log
```

**Configuration**

```bash
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades  # Config avancée
sudo nano /etc/apt/apt.conf.d/20auto-upgrades        # Fréquence
```

**Redémarrage nécessaire ?**

```bash
cat /var/run/reboot-required
cat /var/run/reboot-required.pkgs
```

**Désactiver**

```bash
sudo systemctl stop unattended-upgrades
sudo systemctl disable unattended-upgrades
```

---

## Conclusion

Les mises à jour de sécurité automatiques sont essentielles pour maintenir un système Linux sécurisé sans effort manuel constant. En configurant `unattended-upgrades` avec les bonnes options (sécurité uniquement, blacklist des paquets critiques, notifications email), vous réduisez considérablement la surface d'attaque tout en gardant le contrôle sur les mises à jour sensibles. Surveillez les logs régulièrement et planifiez les redémarrages pour garantir un système stable et à jour.

---

## Voir aussi

- [Installer et configurer Fail2ban sur Ubuntu/Debian]({% post_url 2025-09-21-Installer-et-configurer-Fail2ban-sur-Ubuntu-Debian %})
- [Linux : programmer une tâche avec cron]({% post_url 2025-10-11-Linux-programmer-une-tache-avec-cron %})
- [Comment changer le hostname en ligne de commande sur Ubuntu ou Debian]({% post_url 2025-09-13-Comment-changer-le-hostname-en-ligne-de-commande-sur-Ubuntu-ou-Debian %})
- [Documentation Ubuntu - Mises à jour automatiques](https://help.ubuntu.com/community/AutomaticSecurityUpdates)
- [Documentation Debian - Unattended Upgrades](https://wiki.debian.org/UnattendedUpgrades)
