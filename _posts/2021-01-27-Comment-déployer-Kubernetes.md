---
layout: article
title: "Kubernetes : Comment déployer un cluster k8s bare-metal avec k3s"
tags:
    - kubernetes
    - k8s
    - k3s
    - linux
    - cluster
    - bare-metal
    - sysadmin
    - raspberry-pi
    - tutoriel
author: Pierre Chopinet
---

Dans ce tutoriel, nous allons voir comment déployer un *cluster* kubernetes bare metal, c'est-à-dire sans *cloud provider*.

<!--more-->

## Introduction

Pour ce tutoriel, vous aurez besoin d'un PC, et de plusieurs serveurs que vous voulez mettre en *cluster*. (Ce tutoriel peut fonctionner sur un seul serveur)

Pour déployer notre *cluster,* nous allons utiliser une version allégée de kubernetes nommée [K3s](https://k3s.io), qui est fait pour les appareils ARM comme un *Raspberry PI* ou des serveurs peu puissants. C'est une version simplifiée de k8s avec seulement l'essentiel qui est plus simple à installer et maintenir.

## Préparation des nodes

Sur chacun de vos serveurs, vous allez avoir besoin d'un *user* avec les droits *root*, et de votre clé SSH sur chacun d'entre eux. Il vous faudra sur votre ordinateur le paquet suivant : `openssh-client`, il est par défaut installé sur Ubuntu, mais dans le doute :
```bash
sudo apt install openssh-client
```

Maintenant que vous avez les outils SSH installés sur votre PC, créez une clé SSH avec la commande :

```bash
ssh-keygen -b 4096
```
Laissez l'emplacement du fichier par défaut et ne mettez pas de *passphrase*.
Vous devriez avoir quelque chose dans ce genre :

```bash
Generating public/private rsa key pair.
Enter file in which to save the key (/home/pierre/.ssh/id_rsa):
Created directory '/home/pierre/.ssh'.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/pierre/.ssh/id_rsa
Your public key has been saved in /home/pierre/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:JJKJFEFEG/EfzLJZGREZE fuuf@fuuf-jaaj
The key's randomart image is:
+---[RSA 4096]----+
|  E  .==O .      |
|X +.x.@ o        |
|oB +.= O         |
|=oO +.* o        |
|xx.o +oS      g  |
|:.+ o ac.        |
|   + *.o         |
|  i +      x     |
|                 |
+----[SHA256]-----+
```

Enfin, vous devez copier cette clé SSH sur chacun de vos serveurs avec lesquels vous voulez faire un cluster. Pour cela, nous allons utiliser `ssh-copy-id` pour chaque serveur :
```bash
ssh-copy-id user@host
```
Pour tester si la clé SSH a été bien copiée, essayez de vous connecter à ce serveur :

```bash
ssh user@host
```

##  Installation de K3sup

Nous allons utiliser [K3sup](https://github.com/alexellis/k3sup) pour installer notre *cluster*, c'est un utilitaire simple et rapide pour installer et mettre à jour K3s.
Pour l'installer K3sup sur son PC :

```bash
curl -sLS https://get.k3sup.dev | sh
sudo install k3sup /usr/local/bin/
```

## Création du master

Maintenant que K3sup est installé, nous allons créer notre *master*. Choisissez un de vos serveurs qui deviendra le *master* de votre cluster et lancez depuis votre PC la commande suivante, avec l'adresse ip du serveur choisi et l'*user* choisi. Cet *user* doit avoir les droits *root*.

```bash
SERVER_IP=          # IP du master
SERVER_USER=root    # USER pour se connecter sur le master

k3sup install --no-extras --ip $SERVER_IP --user $SERVER_USER
```
Vous devez avoir maintenant un fichier `kubeconfig` à l'endroit où vous avez exécuté ces commandes. Gardez-le précieusement, il vous permettra d'interagir avec votre *cluster*.

## Ajout des nodes

Maintenant, il faut ajouter vos autres serveurs en tant que *nodes* du *cluster*. Pour cela, on utilise toujours k3sup depuis son ordinateur, avec la commande suivante :

```bash
SERVER_IP=          # IP du master
SERVER_USER=root    # USER pour se connecter sur le master
NODE1_IP=           # IP de la 1ère node
NODE1_user=root     # USER pour se connecter sur la 1ère node

k3sup join --ip $NODE1_IP --user $NODE1_user --server-ip $SERVER_IP --server-user $SERVER_USER
```
Effectuez cette étape pour chacune des *nodes* voulues.

## Script final

Voici le script complet pour réinstaller rapidement K3s, ou mettre à jour la version de kubernetes de votre *cluster*.

```bash
#!/bin/bash

SERVER_IP=
SERVER_USER=root
NODE1_IP=
NODE1_user=root
NODE2_IP=
NODE2_user=root

curl -sLS https://get.k3sup.dev | sh
sudo install k3sup /usr/local/bin/

rm k3sup # fichier temporaire

k3sup install --no-extras --ip $SERVER_IP --user $SERVER_USER

k3sup join --ip $NODE1_IP --user $NODE1_user --server-ip $SERVER_IP --server-user $SERVER_USER

k3sup join --ip $NODE2_IP --user $NODE2_user --server-ip $SERVER_IP --server-user $SERVER_USER
```

##  Installation de kubectl et test du cluster

Pour tester si le *cluster* k8s est bien installé et interagir avec celui-ci, nous allons avoir besoin de `kubectl`.

### Pour Debian/Ubuntu

```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubectl
```
Code repris depuis la documentation officielle.

### Pour d'autres OS

Suivez le guide ici : [https://kubernetes.io/fr/docs/tasks/tools/install-kubectl](https://kubernetes.io/fr/docs/tasks/tools/install-kubectl)

### Test du cluster

Maintenant que `kubectl` est installé, nous allons pouvons tester notre *cluster*. Pour cela, il faut définir une variable d'environnement `KUBECONFIG` correspondant à l'emplacement de notre fichier de config kubernetes généré par K3sup.

```bash
export KUBECONFIG=/chemin/vers/votre/kubeconfig
```
Vous pouvez mettre cette commande dans votre fichier `~/.bashrc` pour que la variable soit toujours définie.

Maintenant, si tout s'est bien passé, en exécutant la commande suivante, vous devriez pouvoir accéder à votre *cluster*.
```bash
kubectl get nodes
```
Vous devriez voir quelque chose dans ce genre :
```bash
NAME        STATUS   ROLES    AGE    VERSION
xxxxxxxx1   Ready    <none>     1d   v1.19.7+k3s1
xxxxxxxx2   Ready    master     1d   v1.19.7+k3s1
xxxxxxxx3   Ready    <none>     1d   v1.19.7+k3s1
```

> Voir aussi : [Comment manipuler du JSON en ligne de commande avec jq]({% post_url 2025-09-17-Comment-utiliser-jq %}) — pratique avec `kubectl -o json | jq`.
## La suite

Maintenant que votre *cluster* est installé, vous pouvez commencer à déployer des services. Attention cependant, K3s ne propose pas de *loadbalancer* et de *storageclass* évolués par défaut, il faudra les installer vous-même. Un article expliquant comment faire devrait sortir prochainement.

## Sources

- [https://github.com/alexellis/k3sup](https://github.com/alexellis/k3sup)
- [https://k3s.io](https://k3s.io/)
- [https://github.com/k3s-io/k3s](https://github.com/k3s-io/k3s)
- [Https://kubernetes.io/fr/docs/tasks/tools/install-kubectl/](https://kubernetes.io/fr/docs/tasks/tools/install-kubectl/)
