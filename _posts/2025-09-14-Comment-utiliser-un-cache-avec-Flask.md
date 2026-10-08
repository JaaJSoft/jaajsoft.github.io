---
layout: article
title: "Comment ajouter un cache à une application Flask"
description: "Ajouter un cache à une application Flask avec Flask-Caching : SimpleCache, clés par utilisateur, invalidation, puis passage en production avec Redis."
tags:
  - python
  - flask
  - http
  - cache
  - redis
  - api
  - rest
  - performance
author: Pierre Chopinet
---

Dans ce tutoriel, nous allons ajouter un cache à une application Flask avec l'extension Flask-Caching, pour répondre plus vite et soulager la base de données ou les API appelées derrière. On commence par un cache en mémoire pour comprendre le fonctionnement, puis on passe à Redis pour la production.
<!--more-->

Dans cet article :
- Installation de Flask-Caching
- Un premier cache en mémoire avec SimpleCache
- Une clé de cache par utilisateur
- Invalider le cache
- Passer en production avec Redis
- Lire et écrire directement dans le cache
- Exemple complet avec Redis

Pré-requis : être à l'aise avec Flask. Si vous débutez, lisez d'abord [Python : Comment faire une api web avec Flask]({% post_url 2021-04-20-Comment-faire-une-api-web-en-python %}).

## Installation de Flask-Caching

```bash
pip install Flask-Caching
```

Les exemples de cet article ont été testés avec Flask 3.1.3 et Flask-Caching 2.5.1.

## Un premier cache en mémoire avec SimpleCache

Le backend `SimpleCache` garde les données en mémoire, dans le processus Python. Il ne demande aucune installation, ce qui en fait un bon point de départ pour voir comment fonctionne l'extension.

Attention, avec Gunicorn et plusieurs *workers*, chaque *worker* est un processus séparé qui a son propre cache : une même page est calculée une fois par *worker*, et une invalidation faite par l'un d'eux ne touche pas les autres. Ce n'est pas gênant en développement, mais en production on lui préfère Redis.

### Initialisation

```python
from flask import Flask
from flask_caching import Cache

app = Flask(__name__)
# Type "SimpleCache": en mémoire, non partagé entre processus
app.config["CACHE_TYPE"] = "SimpleCache"
app.config["CACHE_DEFAULT_TIMEOUT"] = 300  # 5 minutes
cache = Cache(app)
```

`CACHE_DEFAULT_TIMEOUT` est la durée de vie par défaut d'une entrée, en secondes. Chaque décorateur peut la changer avec son paramètre `timeout`.

### Mettre en cache une route

```python
# Met en cache toute la vue pendant 60 secondes
@app.route("/time")
@cache.cached(timeout=60)
def server_time():
    import time
    return {"server_time": time.time()}
```

Pendant 60 secondes, tous les appels à `/time` renvoient la même valeur, sans exécuter la fonction. Notez l'ordre des décorateurs : `@app.route` en premier, puis `@cache.cached`.

### Tenir compte de la query string

Par défaut, la clé de cache est construite à partir du chemin de la requête : `/search?q=flask` et `/search?q=django` partageraient la même entrée. Avec `query_string=True`, les paramètres de l'URL font partie de la clé :

```python
# Met en cache en fonction de la query string (?page=, ?q=, ...)
from flask import request
from time import sleep

@app.route("/search")
@cache.cached(timeout=120, query_string=True)
def search():
    # Simule une opération coûteuse
    sleep(1)
    # On renvoie simplement les paramètres pour l'exemple
    return {"q": request.args.get("q"), "page": request.args.get("page", 1)}
```

Le premier appel à `/search?q=flask&page=2` prend une seconde, les suivants sont immédiats. Flask-Caching trie les paramètres avant de calculer la clé : `/search?page=2&q=flask` tombe sur la même entrée.

### Mémoriser une fonction coûteuse avec memoize

`@cache.cached` met en cache la réponse d'une vue. Pour mettre en cache le résultat d'une fonction Python selon ses arguments, on utilise `@cache.memoize` :

```python
@cache.memoize(timeout=300)
def compute_stats(user_id: int):
    # Ici, faites des requêtes SQL lourdes, appels API, etc.
    import time
    time.sleep(2)
    return {"user_id": user_id, "score": 42}
```

On l'appelle ensuite normalement, par exemple depuis une route :

```python
@app.route("/users/<int:user_id>/stats")
def user_stats(user_id: int):
    return compute_stats(user_id)
```

Le premier appel à `/users/123/stats` prend deux secondes, les suivants sont immédiats. `compute_stats(456)` a sa propre entrée dans le cache.

## Une clé de cache par utilisateur

Une page personnalisée ne doit pas être servie à tout le monde : si la clé de cache ne contient pas l'identité de l'utilisateur, le premier visiteur voit sa page et tous les suivants voient la même. On peut passer à `key_prefix` une fonction qui construit la clé, ici avec l'identifiant de l'utilisateur connecté.

L'exemple s'appuie sur [Flask-Login](https://flask-login.readthedocs.io/) pour accéder à `current_user`. Il faut donc l'avoir installé (`pip install flask-login`) et configuré dans votre application.

```python
from flask_login import current_user

@app.route("/dashboard")
@cache.cached(timeout=120, key_prefix=lambda: f"dashboard:{getattr(current_user, 'id', 'anon')}")
def dashboard():
    # Rendu personnalisé
    return {"hello": getattr(current_user, 'id', 'anon')}
```

L'utilisateur 123 obtient la clé `dashboard:123`, et les visiteurs non connectés partagent la clé `dashboard:anon`.

Le même problème se pose quand la réponse dépend d'un en-tête, la langue par exemple : `query_string=True` ne regarde que l'URL, il faut donc ajouter la langue à la clé soi-même :

```python
from flask import request
from urllib.parse import urlencode
import hashlib

def qs_key(prefix: str = "view"):
    # Clé stable : langue + paramètres triés (ex: items:fr:3f2a...)
    lang = request.accept_languages.best_match(["fr", "en"]) or "fr"
    args = urlencode(sorted(request.args.items(multi=True)))
    h = hashlib.sha1(args.encode()).hexdigest()
    return f"{prefix}:{lang}:{h}"

@app.route("/items")
@cache.cached(timeout=120, key_prefix=lambda: qs_key("items"))
def list_items():
    # ... charge la liste filtrée/paginée ...
    return {"items": [1, 2, 3]}
```

Comme avec `query_string=True`, les paramètres sont triés avant d'être hachés, pour que `?q=flask&page=2` et `?page=2&q=flask` donnent la même clé.

## Invalider le cache

Quand une donnée change, il faut supprimer l'entrée correspondante pour ne pas servir une version périmée jusqu'à la fin du TTL. Flask-Caching propose trois niveaux.

Supprimer une clé précise :

```python
cache.delete("dashboard:123")
```

Invalider une fonction mémorisée (memoize) :

```python
# Supprime tous les caches de compute_stats, tous arguments confondus
cache.delete_memoized(compute_stats)

# Ou seulement pour un argument précis
cache.delete_memoized(compute_stats, 123)
```

Tout vider :

```python
cache.clear()
```

Le bon moment pour invalider est juste après l'écriture en base qui modifie la donnée : après un `POST /users/123`, on appelle `cache.delete_memoized(compute_stats, 123)`. Évitez de vider tout le cache à chaque modification, une invalidation ciblée suffit et évite de tout recalculer.

## Passer en production avec Redis

En production, on utilise un backend partagé comme Redis : tous les *workers*, et toutes les machines, lisent et écrivent dans le même cache.

Installation :

```bash
# Redis côté serveur (exemple Ubuntu/Debian)
sudo apt-get install redis-server
# Client Python
pip install redis
```

Configuration de Flask-Caching pour Redis :

```python
from flask import Flask
from flask_caching import Cache

app = Flask(__name__)
app.config.update(
    CACHE_TYPE="RedisCache",
    CACHE_REDIS_HOST="localhost",  # ou le hostname du service Redis
    CACHE_REDIS_PORT=6379,
    CACHE_REDIS_DB=0,
    CACHE_REDIS_PASSWORD=None,      # définissez un mot de passe si nécessaire
    CACHE_DEFAULT_TIMEOUT=300,
)
cache = Cache(app)
```

Le reste du code (décorateurs, invalidation) ne change pas. Avec docker compose, `CACHE_REDIS_HOST` prend le nom du service Redis (`redis` par exemple).

## Lire et écrire directement dans le cache

En dehors des décorateurs, l'objet `cache` s'utilise comme un dictionnaire avec une durée de vie :

```python
import json
cache.set("key", json.dumps({"x": 1}), timeout=60)
value = json.loads(cache.get("key") or "null")
```

`cache.get()` renvoie `None` si la clé n'existe pas ou a expiré. Passer par JSON n'est pas obligatoire : le backend Redis sérialise les valeurs avec pickle, on peut donc y ranger directement un dictionnaire ou tout objet Python sérialisable. Le JSON a surtout un intérêt si un programme écrit dans un autre langage doit lire le même cache.

Pour le TTL, tout dépend de la fraîcheur attendue : quelques dizaines de secondes pour une liste paginée qui bouge souvent, plusieurs minutes, voire une heure, pour des données qui changent rarement.

## Exemple complet avec Redis

Pour finir, une petite API qui met en cache la lecture d'un produit et invalide l'entrée quand le produit est modifié :

```python
from flask import Flask
from flask_caching import Cache

app = Flask(__name__)
app.config.update(
    CACHE_TYPE="RedisCache",
    CACHE_REDIS_HOST="redis",  # docker compose service name
    CACHE_REDIS_PORT=6379,
    CACHE_DEFAULT_TIMEOUT=300,
)
cache = Cache(app)

@cache.memoize(timeout=120)
def get_product(pid: int):
    # Simule un accès BDD lourd
    import time
    time.sleep(1)
    return {"id": pid, "name": f"Product {pid}"}

@app.route("/products/<int:pid>")
def product(pid: int):
    data = get_product(pid)
    return data

@app.route("/products/<int:pid>", methods=["PUT"])
def update_product(pid: int):
    # ... update BDD ...
    cache.delete_memoized(get_product, pid)  # invalider le cache du produit
    return {"status": "updated", "id": pid}

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

Le premier `GET /products/7` prend une seconde, les suivants sont immédiats. Après un `PUT /products/7`, le `GET` suivant recharge le produit.

## Voir aussi

- [Python : Comment faire une api web avec Flask]({% post_url 2021-04-20-Comment-faire-une-api-web-en-python %})
- [Python : Mettre en cache des fonctions avec lru_cache]({% post_url 2026-05-25-Python-lru_cache %})
- [Ajouter un cache à notre application FastAPI avec redis]({% post_url 2025-08-18-Utiliser-fastapi-cache2-avec-FastAPI %})
- [Comment ajouter du cache à une application Django]({% post_url 2025-11-01-Comment-ajouter-du-cache-a-une-application-Django %})
- [Documentation de Flask-Caching](https://flask-caching.readthedocs.io/)
- [Documentation de Redis](https://redis.io/docs/)
