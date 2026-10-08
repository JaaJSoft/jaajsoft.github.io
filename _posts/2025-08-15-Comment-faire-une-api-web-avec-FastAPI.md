---
layout: article
title: "Python : Comment faire une api web avec FastAPI"
description: "Créer une API web en Python avec FastAPI : premier endpoint, routing, méthodes HTTP et traitement d'une requête POST, avec le code complet du tutoriel."
tags:
- python
- http
- api
- fastapi
- rest
author: Pierre Chopinet
---

Dans ce tutoriel, vous allez apprendre à faire une api web en python avec le
framework FastAPI. <!--more-->
FastAPI est un framework python permettant de réaliser des api web. Il
s'appuie sur les annotations de type de python pour convertir et valider les
données reçues, et génère tout seul la documentation de l'api.

L'objectif de ce tutoriel est d'apprendre à :

- faire une api web en python avec FastAPI
- traiter les requêtes

## Installation

Pour commencer, il vous faut un interpréteur python en version 3.10 ou plus
récente, c'est le minimum demandé par les versions actuelles de FastAPI. Les
exemples de ce tutoriel ont été testés avec Python 3.13 et FastAPI 0.142.

### Linux - Ubuntu (& toutes distributions utilisant APT comme gestionnaire de paquets)

Sous linux, c'est assez simple.

Depuis un terminal, installation de python3 :

```bash
sudo apt install python3
```

Vous aurez ensuite besoin de pip, le gestionnaire de paquets de python. Il est
souvent préinstallé avec python, mais dans le doute :

```bash
sudo apt install python3-pip
```

Maintenant installons FastAPI et un serveur ASGI (uvicorn) :

```bash
pip3 install fastapi uvicorn
```

Sur les distributions récentes (Ubuntu 24.04 par exemple), pip refuse
d'installer des paquets dans le python du système et affiche l'erreur
`externally-managed-environment`. Dans ce cas, créez un environnement virtuel
dans le dossier de votre projet et installez FastAPI dedans :

```bash
sudo apt install python3-venv
python3 -m venv venv
source venv/bin/activate
pip install fastapi uvicorn
```

### Windows

Sur Windows, ça se complique un peu, commencez par télécharger python3 pour
Windows [ici](https://www.python.org/downloads/) et installez-le.

Déplacez-vous dans le dossier où vous avez installé python et faites :

`shift + click droit -> ouvrir une fenêtre powershell` (sur Windows 7, pour les
réfractaires au changement, ça doit être cmd)

Vous êtes normalement dans un terminal, entrez alors :

```powershell
.\python.exe -m pip install fastapi uvicorn
```

### MacOS

N'ayant pas de Mac, je ne peux pas tester l'installation. Il faut toutefois
aussi utiliser python et [pip](https://pypi.org/project/pip/), et suivre les
instructions pour linux afin d'installer FastAPI et uvicorn.

## Une requête HTTP ?

> L'***HyperText Transfer Protocol*** (**HTTP**, littéralement « protocole de
> transfert [hypertexte](https://fr.wikipedia.org/wiki/Hypertexte) ») est
> un [protocole de communication](https://fr.wikipedia.org/wiki/Protocole_de_communication) [client-serveur](https://fr.wikipedia.org/wiki/Client-serveur)
> développé pour le
*[World Wide Web](https://fr.wikipedia.org/wiki/World_Wide_Web)*.

Source Wikipédia.

Il existe 5 principales méthodes HTTP :

- GET : accéder à une ressource
- HEAD : récupérer l'en-tête d'une ressource, par exemple pour connaître la date
  de sa dernière modification (utile pour le système de cache d'un navigateur)
- POST : ajouter une ressource
- PUT : mettre à jour une ressource
- DELETE : supprimer une ressource

## Qu'est-ce qu'une API web ?

> Une API Web est une interface de programmation composée d'un ou de plusieurs
> endpoints exposés publiquement via le Web, le plus souvent au moyen d'un
> système basé sur serveur web HTTP.

Source Wikipédia.

À ne pas confondre avec une API REST, qui est une api web avec un ensemble de
contraintes et de règles prédéfinies à respecter. Toutes les API web ne sont pas
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

Si uvicorn n'est pas trouvé, vous pouvez essayer de lancer :

```bash
python -m uvicorn app:app --reload
```

Si vous allez sur `http://127.0.0.1:8000/` avec votre navigateur web, vous
devriez avoir :

```
"Hello World"
```

Ou alors avec `curl` :

```bash
curl http://127.0.0.1:8000/
"Hello World"
```

Les guillemets sont normaux : FastAPI convertit ce que retourne la fonction en
JSON, et en JSON une chaîne de caractères s'écrit entre guillemets.

FastAPI génère aussi une documentation interactive de l'api, sans rien
ajouter : elle est accessible sur `http://127.0.0.1:8000/docs` (Swagger UI) et
sur `http://127.0.0.1:8000/redoc` (ReDoc).

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
"test_endpoint"
```

### Passer des paramètres

Dans la vraie vie, il est parfois (même très souvent) nécessaire de passer des
paramètres à notre _endpoint_.
Pour passer des paramètres avec le *routing*, on utilise les `{}` dans le chemin
et on déclare la variable en paramètre de la fonction, avec son type :

```python
@app.get('/test/{id_test}')
def test_endpoint(id_test: str):
    return 'test ' + id_test
```

Ce qui retourne :

```bash
curl http://127.0.0.1:8000/test/1
"test 1"
```

Si on annote le paramètre avec le type `int`, FastAPI le convertit et le valide
automatiquement :

```python
@app.get('/test/{id_test}')
def test_endpoint(id_test: int):
    return f'test {id_test}'
```

Un appel avec autre chose qu'un entier est alors refusé avec une erreur 422 :

```bash
curl http://127.0.0.1:8000/test/abc
{"detail":[{"type":"int_parsing","loc":["path","id_test"],"msg":"Input should be a valid integer, unable to parse string as an integer","input":"abc"}]}
```

Quelques types utiles pris en charge nativement (via annotations Python) :

- str, int, float, bool
- UUID (`from uuid import UUID`)
- datetime, date, time (`from datetime import datetime` ...)

Il est également possible d'utiliser des paramètres de requête (query params) :
FastAPI considère comme tels les paramètres de type simple (str, int...) qui
n'apparaissent pas dans le chemin.

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

Pour filtrer ou mettre en forme ces réponses JSON dans le terminal, on peut
envoyer la sortie de curl dans jq (voir l'article
[Comment manipuler du JSON en ligne de commande avec jq]({% post_url 2025-09-17-Comment-utiliser-jq %})).

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
"test_endpoint_get"
```

Le GET renvoie bien la bonne valeur, mais si on tente avec un POST ça ne
fonctionne pas ! FastAPI répond alors avec une erreur `405 Method Not Allowed`,
car aucune route POST n'est déclarée sur ce chemin :

```bash
curl -X POST http://127.0.0.1:8000/test
{"detail":"Method Not Allowed"}
```

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
FastAPI convertit automatiquement le JSON reçu en objet Python (le modèle
Pydantic), et l'objet retourné en JSON.

```bash
curl -X POST http://127.0.0.1:8000/test \
  -H "Content-Type: application/json" \
  -d "{\"param1\":\"jeej\"}"
{"param1":"jeej"}
```

Pour envoyer des fichiers dans un POST (`multipart/form-data`), il faut s'y
prendre autrement : c'est le sujet de l'article
[sur l'upload de fichiers avec FastAPI]({% post_url 2025-08-30-Comment-envoyer-des-fichiers-avec-FastAPI %}).

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
performante.

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

@app.get('/items')
def list_items(q: Optional[str] = None, limit: int = 10):
    return {"q": q, "limit": limit}

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
- [Comment dockeriser une application FastAPI]({% post_url 2025-08-16-Comment-dockeriser-une-api-web-avec-FastAPI %})
- [Organiser une application FastAPI en plusieurs fichiers]({% post_url 2025-08-17-Organiser-une-application-FastAPI-en-plusieurs-fichiers %})
- [Ajouter un cache à notre application FastAPI avec redis]({% post_url 2025-08-18-Utiliser-fastapi-cache2-avec-FastAPI %})
- [Comment ajouter un rate limiter à notre application FastAPI avec redis]({% post_url 2025-09-20-Limiter-le-rate-d-une-API-FastAPI-avec-Redis %})
- [Python : Comment faire une api web avec Flask]({% post_url 2021-04-20-Comment-faire-une-api-web-en-python %})
- [La doc de FastAPI](https://fastapi.tiangolo.com/)
