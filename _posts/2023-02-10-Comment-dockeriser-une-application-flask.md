---
layout: article
title: "Comment Dockeriser une application flask"
author: Pierre Chopinet
tags:

- python
- flask
- docker
- tutoriel

---

Dans ce tutoriel, nous allons apprendre comment _dockeriser_ son api _flask_
avec _docker_ et _gunicorn_.
<!--more-->

L'objectif de ce tutoriel est d'apprendre comment :

- Apprendre les bases de _docker_
- _Dockeriser_ son api _flask_

## Flask

Pré-requis, savoir développer une application _flask_ "simple". Si ce n'est pas le
cas n'hésitez pas à aller voir notre tutoriel sur le sujet :

[Python : Comment faire une api web avec Flask](https://blog.jaaj.dev/2021/04/20/Comment-faire-une-api-web-en-python.html)

## Docker

### Docker qu'est-ce que c'est ?

D'après [Wikipedia](https://fr.wikipedia.org/wiki/Docker_(logiciel)) :
> Docker est une plateforme permettant de lancer certaines applications dans des
> conteneurs logiciels. « Docker est un outil qui peut empaqueter une
> application
> et ses dépendances dans un conteneur isolé, qui pourra être exécuté sur
> n'importe quel serveur ». Il ne s'agit pas de virtualisation, mais de
> conteneurisation, une forme plus légère qui s'appuie sur certaines parties de
> la
> machine hôte pour son fonctionnement. Cette approche permet d'accroître la
> flexibilité et la portabilité d’exécution d'une application, laquelle va
> pouvoir
> tourner de façon fiable et prévisible sur une grande variété de machines
> hôtes,
> que ce soit sur la machine locale, un cloud privé ou public, une machine nue,
> etc.

Autrement dit : _Docker_ permet de faire abstraction de son OS, et de pouvoir
partager le même environnement entre sa machine de développement et son/ses
serveurs en production.

## Gunicorn

Par défaut, le serveur utilisé par flask est dédié au développement en local de
son application, et n'est pas adapté à être utilisé en production. Il faut
utiliser un autre serveur web HTTP pour gérer les requêtes à notre application.
_Gunicorn_ est l'un des serveurs web HTTP les plus simples et les plus légers pour
_flask_.

On installe _gunicorn_ en local à l'aide de _pip_ :
```bash
pip3 install gunicorn
```

Pour lancer son application :
```bash
#!/bin/sh

gunicorn -w 4 "app:app" -b 0.0.0.0:8000 -t 0
```

- Le premier `app` correspond au nom du fichier d'entrée de votre application
  flask
- Le deuxieme `app` correspond à la variable définie dans le fichier
  précédent (`app = Flask(__name__)`)

Enregistrez cette commande dans un script _bash_ nommé `run.sh` à la racine de
votre projet.

Pour la suite du tutoriel, il est aussi nécessaire d'avoir un fichier `requirement.txt` à la racine de son projet avec les dépendances nécessaires au bon fonctionnement de son application _flask_. Au minimum, on a besoin de :
```
flask~=2.1.2
gunicorn
```
Maintenant tout doit être bon du côté de _python_, on attaque _docker_ !

## Création de notre image docker

Pour créer notre conteneur docker, nous avons besoin de définir comment
construire une image _docker_. Pour cela nous allons utiliser un fichier
nommé `dockerfile`.

Un Dockerfile est un fichier texte qui contient toutes les commandes à exécuter
pour construire une image.
Créez ce fichier à la racine de votre projet.

Nous allons baser notre image docker sur _Alpine_, une distribution légère dédiée
au
conteneur docker. Cette distribution permet de réduire sensiblement la taille
des images _docker_.

```dockerfile
FROM alpine
```

On choisit l'emplacement de notre application (nommé le dossier de travail par
la suite), à partir de la racine du système de fichier virtuel du conteneur :

```dockerfile
WORKDIR /app
```

On installe ensuite _python 3_ et les différentes dépendances nécessaires pour
_flask_ :

```dockerfile
RUN apk add --update --no-cache python3 py3-pip gcc musl-dev python3-dev libffi-dev openssl-dev
RUN python3 -m ensurepip # permet d'être sur que pip est bien présent avec python
```

On copie le code de notre application _python_ vers le dossier de travail défini
précédemment :

```dockerfile
COPY . .
```

Par défaut cette commande va copier l'ensemble de votre repertoire local dans le
dossier de travail du conteneur ! Or certains fichiers ne sont pas forcément
nécessaires dans notre _docker_ et même pour certains fichiers sont dangereux
d'avoir dans le conteneur, comme des fichiers de _CI_ avec des _tokens_ de
déploiement.

Pour régler ce petit problème, il est possible comme avec _git_ de créer un
fichier pour blacklister des fichiers. Au meme niveau que votre _Dockerfile_,
créez un fichier nommé : `.dockerignore`

Ajoutez dedans tous les fichiers à ne pas inclure dans le conteneur, dans mon
cas je retire mes fichiers relatifs à _git_ :

```
.gitignore
.git
README.md
```

Enfin j'installe les dépendances définies précédemment :

```dockerfile
RUN pip3 install -r requirements.txt
```

Je rends executable notre script pour lancer _gunicorn_ et notre application :

```dockerfile
RUN chmod +x run.sh
```

Cette commande définie quel port va être exposé par notre _docker_, ici on met
donc le port sur lequel _gunicorn_ est lancé, le 8000.

```dockerfile
EXPOSE 8000
```

Enfin j'indique quelle commande lancer au démarrage de notre conteneur, dans
notre cas, notre script de lancement.

```dockerfile
CMD ["./run.sh"]
```

Et voilà notre dockerfile est terminé.

## Le dockerfile complet :

```dockerfile
FROM alpine

WORKDIR /app

RUN apk add --update --no-cache python3 py3-pip gcc musl-dev python3-dev libffi-dev openssl-dev
RUN python3 -m ensurepip

COPY . .
RUN pip3 install -r requirements.txt

RUN chmod +x run.sh

EXPOSE 8000

CMD ["./run.sh"]
```

## Test de notre docker

Maintenant, il va falloir _build_ et tester notre image

```bash
docker build -t mon_app .
```

On lance notre application sur le port 5000 de notre OS en le mappant sur le
8000 de notre conteneur avec la commande suivante :

```bash
docker run -p 5000:8000 mon_app
```

Pour tester que tout marche bien, on teste une des routes flask définie dans le
tutoriel précédent :

```bash
curl -X GET http://127.0.0.1:5000/test
test_endpoint_get
```

## Conclusion

Dans ce tutoriel, vous aurez appris à conteneuriser votre application python et
à
la rendre prête pour être mise en production. Il ne reste plus qu'à la publier
et/ou à la déployer quelque part.

## Voir aussi

- [Documentation Docker](https://docs.docker.com/get-started/)
- [Python : Comment faire une api web avec Flask](https://blog.jaaj.dev/2021/04/20/Comment-faire-une-api-web-en-python.html)
- [Python : Comment faire des requêtes HTTP avec requests](https://blog.jaaj.dev/2020/05/22/Comment-faire-des-requetes-http-en-python-avec-requests.html)
