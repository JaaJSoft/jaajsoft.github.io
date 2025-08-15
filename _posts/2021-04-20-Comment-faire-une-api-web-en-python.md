---
layout: article
title: "Python : Comment faire une api web avec Flask"
tags:
- python
- http
- api
- tutoriel
- flask
- rest
author: Pierre Chopinet
---

Dans ce tutoriel, vous allez apprendre à faire une api web en python avec le
framework Flask. <!--more-->
Le framework flask est un framework python permettant la réalisation d'un site
web ou d'une api web. Son principal avantage est d'être simple à utiliser, mais
sans perdre de fonctionnalités, de plus il peut quasiment tout faire grâce à de
nombreuses extensions.

L'objectif de ce tutoriel est d'apprendre comment faire :

- Une api web en python
- Le traitement des requêtes

## Installation

Pour commencer, il vous faut un interpréteur python en version 3, dans mon cas,
j'utiliserai python 3.8

### Linux - Ubuntu (& toutes distributions utilisant APT comme gestionnaire de paquets)

Sous linux, c'est assez simple.

Depuis un terminal, installation de python3 :

```bash
sudo apt install python3
```

Vous aurez ensuite besoin de pip le gestionnaire de package de python, il est
souvent préinstallé avec python, mais dans le doute :

```bash
sudo apt install python3-pip
```

Maintenant installons *flask* :

```bash
pip3 install flask
```

Si vous avez une erreur vous disant que vous n'avez pas assez de permissions,
faites :

```bash
pip3 install --user flask
```

### Windows

Sur Windows, ça se complique un peu, commencez par télécharger python3 pour
Windows [ici](https://www.python.org/downloads/) et installez-le.

Déplacez-vous dans le dossier où vous avez installé python et faites :

`shift + click droit -> ouvrir une fenêtre powershell` (sur Windows 7 pour les
réfractaires au changement ça doit être cmd)

Vous êtes normalement dans un terminal, entrez alors :

```powershell
.\python.exe -m pip install flask
```

### MacOS

N'ayant pas de Mac, je ne peux pas tester l'installation, il faut toutefois
aussi utiliser python et [PIP](https://pypi.org/project/pip/), et suivre les
instructions pour linux afin d'installer *flask*.

## Une requête HTTP ?

> L'***HyperText Transfer Protocol*** (**HTTP**, littéralement « protocole de
> transfert [hypertexte](https://fr.wikipedia.org/wiki/Hypertexte) ») est
> un [protocole de communication](https://fr.wikipedia.org/wiki/Protocole_de_communication) [client-serveur](https://fr.wikipedia.org/wiki/Client-serveur)
> développé pour le
*[World Wide Web](https://fr.wikipedia.org/wiki/World_Wide_Web)*.

Source Wikipédia.

Il existe 5 principales requêtes HTTP :

- GET, permet d'accéder à une ressource.
- HEAD, permet de récupérer l'entête d'une ressource, pour par exemple connaitre
  la date de sa dernière modification (utile pour le système de cache d'un
  navigateur)
- POST, permet d'ajouter une ressource
- PUT, permet de mettre à jour une ressource
- DELETE, permet de supprimer une ressource

## Qu'est-ce qu'une API web ?

> Une API Web est une interface de programmation composée d'un ou de plusieurs
> points endpoints exposés publiquement via le Web, le plus souvent au moyen d'un
> système basé sur serveur web HTTP.

Source Wikipédia.

À ne pas confondre avec une API REST, qui est une api web avec un ensemble de
contraintes et de règles prédéfinies à utiliser. Toutes les API web ne sont pas
des API REST...

## Un premier *Endpoint*

Créez un fichier `app.py` avec le contenu suivant :

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def super_endpoint():
    return 'Hello World'
```

Pour lancer votre premier *Endpoint* :

```bash
flask run
```

Ou sinon :

```bash
python -m flask run
```

Si vous allez sur `http://127.0.0.1:5000/` avec votre navigateur web, vous
devriez avoir :

```
Hello World
```

Ou alors avec `curl`

```bash
curl http://127.0.0.1:5000/
Hello World
```

Super, nous avons notre premier "hello world", mais comment faire pour avoir
plusieurs routes possibles ?

## *Routing*

Pour régler ce problème, nous allons utiliser une fonctionnalité intégrée à
Flask pour faire du _routing_ par rapport à notre URL.
On crée un nouvel *endpoint* qu'on pourra appeler avec
l'URL : `http://127.0.0.1:5000/test`

```python
@app.route('/test')
def test_endpoint():
    return 'test_endpoint'
```

```bash
curl http://127.0.0.1:5000/test
test_endpoint
```

### Passer des paramètres

Dans la vraie vie, il est parfois (même très souvent) nécessaire de passer des
paramètres à notre _endpoint_.
Pour passer des paramètres avec le *routing* on utilise les `<>` avec un simple
argument de notre fonction

```python
@app.route('/test/<id_test>')
def test_endpoint(id_test):
    return 'test ' + id_test
```

Ce qui retourne :

```bash
curl http://127.0.0.1:5000/test/1
test 1
```

Par défaut le type est un string, mais il est possible de forcer le *cast* vers
un type en ajoutant un type dans les `<>` de la route :

``` python
@app.route('/test/<int:id_test>')
def test_endpoint(id_test):
    return 'test ' + id_test
```

Il existe différents convertisseurs intégrés dans flask :

- string
- int
- float
- path
- uuid

Il est également possible de créer ses propres convertisseurs si nécessaire.

## Méthodes HTTP

Pour le moment notre API répond à tous les types de requêtes HTTP ce qui peut
poser des problèmes, pour spécifier pour quelles méthodes l'*endpoint* doit être
disponible, on ajoute dans l'annotation `@app.route` un nouveau
paramètre `methods`

```python
@app.route('/test', methods=["GET"])
def test_endpoint_get():
    return 'test_endpoint_get'
```

```bash
curl -X GET http://127.0.0.1:5000/test
test_endpoint_get
```
Le GET avec l'aide curl renvoie bien la bonne valeur, mais si on tente avec un POST ça ne fonctionne pas ! Car _Flask_ ne sait pas vers quoi rediriger notre requête HTTP.

```bash
curl -X POST http://127.0.0.1:5000/test
```

## Traiter une requête POST
Pour traiter une requête POST, on a besoin de récupérer les données de celle-ci.
On importe donc `request` pour cela.

```python
from flask import request

@app.route('/test', methods=["POST"])
def test_endpoint_post():
    data = request.form
    # Traiter la requête
    return data
```
Le module _request_ converti automatiquement les données de la requête en dictionnaire python.
Dans l'autre sens si notre API retourne un dictionnaire, flask se charge de le convertir pour nous en json :
```bash
curl -X POST http://127.0.0.1:5000/test -d "param1=jeej"
{"param1":"jeej"}
```

### Exemple d'un POST avec un traitement simpliste

```python
@app.route('/exemple', methods=["POST"])
def test2_endpoint_post():
    """
    Exemple de traitement
    """
    responses = {}
    param1 = request.form["param1"]
    responses["return1"] = param1 + "AAA"
    return responses
```

```bash
curl -X POST http://127.0.0.1:5000/exemple -d "param1=jeej"
{"return1":"jeejAAA"}
```

Voilà, vous êtes maintenant capable de créer une api web simple, mais
performante. D'autres tutoriels sur flask seront prochainement disponibles : par exemple pour
interroger une base de données et avoir des données dynamiques ou docker son application flask.

## Le code complet de ce tutoriel

```python
from flask import Flask
from flask import request

app = Flask(__name__)

@app.route('/')
def super_endpoint():
    return 'Hello World'

@app.route('/test/<id_test>')
def test_endpoint(id_test):
    return 'test ' + id_test

@app.route('/test', methods=["GET"])
def test_endpoint_get():
    return 'test_endpoint_get'

@app.route('/test', methods=["POST"])
def test_endpoint_post():
    data = request.form
    # traiter la requête
    return data

@app.route('/exemple', methods=["POST"])
def test2_endpoint_post():
    """
    Exemple de traitement
    """
    responses = {}
    param1 = request.form["param1"]
    responses["return1"] = param1 + "AAA"
    return responses

```

## Voir aussi

- [Comment dockeriser une application flask]({% post_url 2023-02-10-Comment-dockeriser-une-application-flask %})
- [Comment faire des requêtes HTTP en python avec requests]({% post_url 2020-05-22-Comment-faire-des-requetes-http-en-python-avec-requests %})
- [Comment créer un bot twitter en python avec tweepy]({% post_url 2019-06-20-Comment-créer-un-bot-twitter-en-python %})
- [Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %})
- [La doc de flask](https://flask.palletsprojects.com/en/1.1.x/)
