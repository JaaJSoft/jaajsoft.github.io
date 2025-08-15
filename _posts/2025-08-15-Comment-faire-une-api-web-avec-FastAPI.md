---
layout: article
title: "Python : Comment faire une api web avec FastAPI"
tags:
- python
- http
- api
- tutoriel
- fastapi
- rest
author: Pierre Chopinet
---

Dans ce tutoriel, vous allez apprendre à faire une api web en python avec le
framework FastAPI. <!--more-->
FastAPI est un framework web moderne pour Python, conçu pour créer des APIs
performantes rapidement, avec de la validation automatique, des schémas
OpenAPI/Swagger et une excellente expérience développeur.

L'objectif de ce tutoriel est d'apprendre comment faire :

- Une api web en python avec FastAPI
- Le traitement des requêtes

## Installation

Pour commencer, il vous faut un interpréteur python en version 3, dans mon cas,
j'utiliserai python 3.10

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

Maintenant installons FastAPI et un serveur ASGI (uvicorn) :

```bash
pip3 install fastapi uvicorn
```

Si vous avez une erreur vous disant que vous n'avez pas assez de permissions,
faites :

```bash
pip3 install --user fastapi uvicorn
```

### Windows

Sur Windows, ça se complique un peu, commencez par télécharger python3 pour
Windows [ici](https://www.python.org/downloads/) et installez-le.

Déplacez-vous dans le dossier où vous avez installé python et faites :

`shift + click droit -> ouvrir une fenêtre powershell` (sur Windows 7 pour les
réfractaires au changement ça doit être cmd)

Vous êtes normalement dans un terminal, entrez alors :

```powershell
.\python.exe -m pip install fastapi uvicorn
```

### MacOS

N'ayant pas de Mac, je ne peux pas tester l'installation, il faut toutefois
aussi utiliser python et [PIP](https://pypi.org/project/pip/), et suivre les
instructions pour linux afin d'installer FastAPI et uvicorn.

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
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def super_endpoint():
    return "Hello World"
```

Pour lancer votre premier *Endpoint* :

```bash
uvicorn app:app --reload
```

Si unicorn n'est pas trouvé vous pouvez essayer de lancer :
```
python -m uvicorn app:app --reload
```

Si vous allez sur `http://127.0.0.1:8000/` avec votre navigateur web, vous
devriez avoir :
```
Hello World
```

Ou alors avec `curl`

```bash
curl http://127.0.0.1:8000/
Hello World
```

Note : FastAPI fournit automatiquement une documentation interactive:

- Swagger UI: http://127.0.0.1:8000/docs
- ReDoc: http://127.0.0.1:8000/redoc

Super, nous avons notre premier "hello world", mais comment faire pour avoir
plusieurs routes possibles ?

## *Routing*

Pour régler ce problème, nous allons utiliser une fonctionnalité intégrée à
FastAPI pour faire du _routing_ par rapport à notre URL.
On crée un nouvel *endpoint* qu'on pourra appeler avec
l'URL : `http://127.0.0.1:8000/test`

```python
@app.get('/test')
def test_endpoint():
    return 'test_endpoint'
```

```bash
curl http://127.0.0.1:8000/test
test_endpoint
```

### Passer des paramètres

Dans la vraie vie, il est parfois (même très souvent) nécessaire de passer des
paramètres à notre _endpoint_.
Pour passer des paramètres avec le *routing* on utilise les `{}` dans le chemin
et on tape les variables en paramètres de fonction :

```python
@app.get('/test/{id_test}')
def test_endpoint(id_test: str):
    return 'test ' + id_test
```

Ce qui retourne :

```bash
curl http://127.0.0.1:8000/test/1
test 1
```

En tapant en `int`, FastAPI validera et convertira automatiquement :

```python
@app.get('/test/{id_test}')
def test_endpoint(id_test: int):
    return f'test {id_test}'
```

Quelques types utiles pris en charge nativement (via annotations Python) :

- str, int, float, bool
- UUID (`from uuid import UUID`)
- datetime, date, time (`from datetime import datetime` ...)

Il est également possible d'utiliser des paramètres de requête (query params) :

```python
from typing import Optional

@app.get('/items')
def list_items(q: Optional[str] = None, limit: int = 10):
    return {"q": q, "limit": limit}
```

```bash
curl "http://127.0.0.1:8000/items?q=abc&limit=5"
{"q":"abc","limit":5}
```

## Méthodes HTTP

Pour spécifier pour quelles méthodes l'*endpoint* doit être disponible, on
utilise le décorateur approprié (`@app.get`, `@app.post`, etc.) :

```python
@app.get('/test')
def test_endpoint_get():
    return 'test_endpoint_get'
```

```bash
curl -X GET http://127.0.0.1:8000/test
test_endpoint_get
```

Si on tente avec un POST sur cet *endpoint*, FastAPI retourne automatiquement
`405 Method Not Allowed`.

## Traiter une requête POST
Pour traiter une requête POST et valider les données, on utilise un modèle
[Pydantic](https://docs.pydantic.dev/).

```python
from pydantic import BaseModel

class Data(BaseModel):
    param1: str

@app.post('/test')
def test_endpoint_post(data: Data):
    # Traiter la requête
    return data
```
FastAPI convertit automatiquement le JSON en objet Python (modèle Pydantic) et
inversement retourne du JSON.

```bash
curl -X POST http://127.0.0.1:8000/test \
  -H "Content-Type: application/json" \
  -d "{\"param1\":\"jeej\"}"
{"param1":"jeej"}
```

### Exemple d'un POST avec un traitement simpliste

```python
@app.post('/exemple')
def test2_endpoint_post(data: Data):
    """
    Exemple de traitement
    """
    responses = {}
    responses["return1"] = data.param1 + "AAA"
    return responses
```

```bash
curl -X POST http://127.0.0.1:8000/exemple \
  -H "Content-Type: application/json" \
  -d "{\"param1\":\"jeej\"}"
{"return1":"jeejAAA"}
```

Voilà, vous êtes maintenant capable de créer une api web simple, mais
performante. D'autres tutoriels sur FastAPI pourront par exemple couvrir la
connexion à une base de données, la gestion des dépendances ou le déploiement en
conteneur.

## Le code complet de ce tutoriel

```python
from fastapi import FastAPI
from pydantic import BaseModel
from typing import Optional

app = FastAPI()

@app.get('/')
def super_endpoint():
    return 'Hello World'

@app.get('/test/{id_test}')
def test_endpoint(id_test: int):
    return f'test {id_test}'

@app.get('/test')
def test_endpoint_get():
    return 'test_endpoint_get'

class Data(BaseModel):
    param1: str

@app.post('/test')
def test_endpoint_post(data: Data):
    # traiter la requête
    return data

@app.post('/exemple')
def test2_endpoint_post(data: Data):
    """
    Exemple de traitement
    """
    responses = {}
    responses['return1'] = data.param1 + 'AAA'
    return responses
```

## Voir aussi
- [Comment faire une api avec flask](https://blog.jaaj.dev/2021/04/20/Comment-faire-une-api-web-en-python.html)
- [Comment dockeriser une application flask](https://blog.jaaj.dev/2023/02/10/Comment-dockeriser-une-application-flask.html)
- [Comment faire des requêtes HTTP en python avec requests](https://blog.jaaj.dev/2020/05/22/Comment-faire-des-requetes-http-en-python-avec-requests.html)
- [La doc de FastAPI](https://fastapi.tiangolo.com/)
