---
layout: article
title: "Python : Comment faire des requêtes HTTP avec requests"
tags:
    - python
    - http
    - requests
    - tutoriel
author: Pierre Chopinet
---

Dans ce tutoriel, vous allez apprendre à faire des requêtes HTTP en Python en utilisant la bibliothèque requests. <!--more--> L'objectif de ce tutoriel est d'apprendre comment faire :

- Faire des requêtes HTTP (GET, POST, PUT, DELETE, HEAD)
- Passer des paramètres d'URL (params), un corps de requête (data vs json) et des headers
- Traiter le résultat d'une requête (texte, JSON, encodage)

## Installation

Pour commencer, il vous faut un interpréteur python en version 3, dans mon cas, j'utiliserai python 3.7.5.

### Linux - Ubuntu (& toutes distributions utilisant APT comme gestionnaire de paquets)

Sous linux, c'est assez simple.

Depuis un terminal, installation de python3 :

```bash
sudo apt install python3
```

Vous aurez ensuite besoin de pip le gestionnaire de package de python, il est souvent préinstallé avec python, mais dans le doute :

```bash
sudo apt install python3-pip
```

Maintenant installons *requests* :

```bash
pip3 install requests
```

Si vous avez une erreur vous disant que vous n'avez pas assez de permissions, faites :

```bash
pip3 install --user requests
```

### Windows

Sur Windows, ça se complique un peu, commencez par télécharger python3 pour Windows [ici](https://www.python.org/downloads/) et installez-le.

Déplacez-vous dans le dossier où vous avez installé python et faites :

`shift + click droit -> ouvrir une fenêtre powershell` (sur Windows 7 pour les réfractaires au changement ça doit être cmd)

Vous êtes normalement dans un terminal, entrez alors :

```powershell
.\python.exe -m pip install requests
```

### MacOS

N'ayant pas de Mac, je ne peux pas tester l'installation, il faut toutefois aussi utiliser python et [PIP](https://pypi.org/project/pip/), et suivre les instructions pour linux afin d'installer la bibliothèque *requests*.

## Une requête HTTP ?

> L'***HyperText Transfer Protocol*** (**HTTP**, littéralement « protocole de transfert [hypertexte](https://fr.wikipedia.org/wiki/Hypertexte) ») est un [protocole de communication](https://fr.wikipedia.org/wiki/Protocole_de_communication) [client-serveur](https://fr.wikipedia.org/wiki/Client-serveur) développé pour le *[World Wide Web](https://fr.wikipedia.org/wiki/World_Wide_Web)*.

Source Wikipédia

Il existe 5 principales requêtes HTTP

- GET, permet d'accéder à une ressource.
- HEAD, permet de récupérer l’en‑tête d’une ressource, par exemple pour connaître la date de sa dernière modification (utile pour le système de cache d’un navigateur)
- POST, permet d'ajouter une ressource
- PUT, permet de mettre à jour une ressource
- DELETE, permet de supprimer une ressource

## Requêtes basiques

### Requête GET

```python
import requests

response = requests.get("https://blog.jaaj.dev")
print(response.text)
```

Cette requête HTTP GET affiche la page HTML correspondante.

### Requête GET avec paramètres

Une requête GET peut avoir des paramètres, par exemple pour `https://blog.jaaj.dev/archive.html?tag=python` :

```python
import requests

params = {"tag": "python"}
response = requests.get("https://blog.jaaj.dev/archive.html", params=params)
print(response.text)
```
### Requête HEAD

*requests* permet d'accéder uniquement aux headers d'une page en utilisant la requête *head* :

```python
import requests

response = requests.head("https://blog.jaaj.dev")
print(response.headers)
```
Ce qui permet d'avoir les informations suivantes sur la ressource :
```text
{'Connection': 'keep-alive', 'Content-Length': '10575', 'Server': 'GitHub.com', 'Content-Type': 'text/html; charset=utf-8', 'Strict-Transport-Security': 'max-age=31556952', 'Last-Modified': 'Fri, 20 Mar 2020 09:39:39 GMT', 'ETag': 'W/"5e748f5b-9528"', 'Access-Control-Allow-Origin': '*', 'Expires': 'Fri, 22 May 2020 09:46:06 GMT', 'Cache-Control': 'max-age=600', 'Content-Encoding': 'gzip', 'X-Proxy-Cache': 'MISS', 'X-GitHub-Request-Id': '7DD0:5D0B:292B1C:3342F8:5EC79D05', 'Accept-Ranges': 'bytes', 'Date': 'Fri, 22 May 2020 09:36:06 GMT', 'Via': '1.1 varnish', 'Age': '0', 'X-Served-By': 'cache-cdg20727-CDG', 'X-Cache': 'MISS', 'X-Cache-Hits': '0', 'X-Timer': 'S1590140167.863279,VS0,VE107', 'Vary': 'Accept-Encoding', 'X-Fastly-Request-ID': '7bccbb14a86614bdc56df3295ea37e17a144569b'}
```
Très peu clair pour un humain, mais cela permet pour un navigateur d'avoir des informations très utiles sur la ressource demandée.

### Requête POST

Et finalement la requête *post* qui s'utilise de la même manière qu'une requête *get* sauf que les paramètres sont passés dans le corps de la requête et pas avec l'url.
Pour passer un json en paramètre dans le _body_ :
```python
import requests

data = {"example": "test"}
response = requests.post("https://blog.jaaj.dev/archive.html", json=data)
print(response.status_code)
```

Explication : `params=` ajoute des paramètres dans l’URL, `data=` envoie un formulaire (application/x-www-form-urlencoded) et `json=` envoie un JSON (application/json). Choisissez le bon champ selon ce que l’API attend.

Exemples :

- Formulaire (data)

```python
import requests

form = {"username": "bob", "password": "secret"}
r = requests.post("https://httpbin.org/post", data=form)
print(r.request.headers["Content-Type"])  # application/x-www-form-urlencoded
```
- Corps JSON (json)

```python
import requests

payload = {"username": "bob"}
r = requests.post("https://httpbin.org/post", json=payload)
print(r.request.headers["Content-Type"])  # application/json
```
- Upload de fichier (multipart/form-data)

```python
import requests

with open("rapport.pdf", "rb") as f:
    r = requests.post("https://httpbin.org/post", files={"file": f})
print(r.status_code)
```

### Requête PUT (mettre à jour une ressource)

```python
import requests

url = "https://jsonplaceholder.typicode.com/posts/1"
payload = {"id": 1, "title": "Mon titre", "body": "Nouveau contenu", "userId": 1}

response = requests.put(url, json=payload)
print(response.status_code)  # ex: 200
print(response.json())
```

> Note : En REST, PUT remplace généralement la ressource entière. Pour une mise à jour partielle, utilisez plutôt PATCH. Selon les API, la réponse peut être 200 (avec un corps JSON) ou 204 No Content.
>
> Explication :
> - PUT est idempotent : répéter la même requête ne change pas l’état après la première.
> - Avec `requests`, `json=` sérialise l’objet Python et ajoute l’en‑tête `Content-Type: application/json` automatiquement. `data=` enverrait un formulaire.
> - Certaines API exigent un contrôle de concurrence optimiste via `ETag`/`If-Match` pour éviter d’écraser des modifications.

### Requête DELETE (supprimer une ressource)

```python
import requests

url = "https://jsonplaceholder.typicode.com/posts/1"
response = requests.delete(url)

print(response.status_code)  # ex: 200 ou 204
print(response.text)         # souvent vide (No Content)
```

> Note : Beaucoup d’API renvoient 204 No Content pour un DELETE réussi.
>
> Explication :
> - DELETE est idempotent : un second appel sur la même ressource renvoie souvent `204` (aucun changement) ou `404` si elle n’existe plus.
> - Le corps de la réponse est souvent vide. Vérifiez `response.status_code` ou utilisez `response.raise_for_status()`.
> - Selon les API, l’opération peut être asynchrone et renvoyer `202 Accepted`.

## Timeout et gestion des erreurs (basique)

Utilisez toujours un `timeout` pour éviter qu’un appel ne bloque indéfiniment, et pensez à `raise_for_status()` pour déclencher une exception en cas d’erreur HTTP (4xx/5xx).

```python
import requests
from requests.exceptions import HTTPError, Timeout, RequestException

try:
    r = requests.get("https://jsonplaceholder.typicode.com/users", timeout=5)
    r.raise_for_status()  # lève HTTPError si 4xx/5xx
    data = r.json()
    print(len(data), "utilisateurs")
except Timeout:
    print("La requête a dépassé le délai (timeout).")
except HTTPError as e:
    print(f"Erreur HTTP: {e.response.status_code}")
except RequestException as e:
    print(f"Erreur réseau: {e}")
```

> Astuce : fixez un `timeout` (ex. 5–10s) sur toutes vos requêtes côté client.

## Traiter le résultat d'une requête vers une API REST

Comme exemple d'API, nous allons utiliser [https://jsonplaceholder.typicode.com](https://jsonplaceholder.typicode.com), une API de test permettant d'expérimenter avec les API REST facilement.

On va faire une requête vers [https://jsonplaceholder.typicode.com/users](https://jsonplaceholder.typicode.com/users) qui renvoie une liste d'utilisateurs au format JSON suivant :
```text
[
  {
    "id": 1,
    "name": "Leanne Graham",
    "username": "Bret",
    "email": "Sincere@april.biz",
    "address": {
      "street": "Kulas Light",
      "suite": "Apt. 556",
      "city": "Gwenborough",
      "zipcode": "92998-3874",
      "geo": {
        "lat": "-37.3159",
        "lng": "81.1496"
      }
    },
    "phone": "1-770-736-8031 x56442",
    "website": "hildegard.org",
    "company": {
      "name": "Romaguera-Crona",
      "catchPhrase": "Multi-layered client-server neural-net",
      "bs": "harness real-time e-markets"
    }
  },
  {
    "id": 2,
    "name": "Ervin Howell",
    "username": "Antonette",
    "email": "Shanna@melissa.tv",
    "address": {
      "street": "Victor Plains",
      "suite": "Suite 879",
      "city": "Wisokyburgh",
      "zipcode": "90566-7771",
      "geo": {
        "lat": "-43.9509",
        "lng": "-34.4618"
      }
    },
    "phone": "010-692-6593 x09125",
    "website": "anastasia.net",
    "company": {
      "name": "Deckow-Crist",
      "catchPhrase": "Proactive didactic contingency",
      "bs": "synergize scalable supply-chains"
    }
  },
  ...
]
```
La bibliothèque *requests* propose un moyen facile de traiter une réponse au format JSON :
```python
import requests

response = requests.get("https://jsonplaceholder.typicode.com/users")
if response.status_code == 200:
    response_json = response.json()

    for user in response_json:
        print(user["name"])
```
Ce code affiche le nom de tous les utilisateurs. On teste si le *status_code* est 200 pour ne traiter le résultat que si la requête est un succès. Il existe plusieurs codes de retour décrits [ici](https://fr.wikipedia.org/wiki/Liste_des_codes_HTTP).

## Changer les headers de la requête

Dans certains cas, il peut être utile de changer les headers d'une requête pour se faire passer pour un navigateur web et accéder à certains contenus dont l'accès est restreint depuis un script.

Par exemple ici pour se faire passer pour Mozilla Firefox :

```python
import requests

headers = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; WOW64; Trident/7.0; rv:11.0) like Gecko'}
response = requests.get("https://jsonplaceholder.typicode.com/users", headers=headers)
```

Pour personnaliser encore plus ses *User-Agent*, il existe une bibliothèque proposant plusieurs *User-Agent* : [fake-useragent](https://pypi.org/project/fake-useragent)

## Télécharger un fichier (streaming)

Pour les fichiers volumineux, activez `stream=True` et écrivez par blocs pour éviter de tout charger en mémoire.

```python
import requests

url = "https://httpbin.org/image/png"
with requests.get(url, stream=True, timeout=10) as r:
    r.raise_for_status()
    with open("image.png", "wb") as f:
        for chunk in r.iter_content(chunk_size=8192):
            if chunk:  # éviter les keep-alive chunks
                f.write(chunk)
```

> Astuce : pour de petits contenus, `response.content` (bytes) suffit ; `response.text` applique un encodage (UTF‑8 par défaut ou `response.encoding`).

## Voir aussi

- Documentation officielle requests: [https://requests.readthedocs.io](https://requests.readthedocs.io/)
- [Python : Comment faire une api web avec Flask]({% post_url 2021-04-20-Comment-faire-une-api-web-en-python %})
- [Python : Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %})
- [Python : Comment sauvegarder des tableaux NumPy]({% post_url 2022-01-25-Comment-sauvegarder-un-tableau-numpy %})
- [Python : Comment utiliser les différents modes d'authentification avec requests]({% post_url 2025-09-05-Comment-utiliser-l-authentification-avec-requests %})
- [Python : Comment utiliser les sessions avec requests pour optimiser vos appels HTTP]({% post_url 2025-09-04-Comment-utiliser-les-sessions-avec-requests %})

