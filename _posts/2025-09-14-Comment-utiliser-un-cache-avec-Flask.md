---
layout: article
title: "Comment utiliser un cache avec Flask"
author: Pierre Chopinet
tags:
  - python
  - flask
  - cache
  - performance
  - tutoriel
---

Dans ce tutoriel, on va voir comment ajouter un cache à une application Flask
pour accélérer les réponses et réduire la charge sur vos bases de données et
API.
<!--more-->

Objectifs de cet article :

- Comprendre les différents types de cache
- Mettre en place Flask-Caching en mémoire pour démarrer rapidement
- Mettre en place un backend Redis pour la production
- Savoir cacher des vues, des fonctions « coûteuses » et gérer l’invalidation
- Éviter les pièges courants (clés, utilisateurs, workers Gunicorn)

> Pré‑requis : si vous débutez avec Flask, lisez d’abord [Python : Comment faire une api web avec Flask]({% post_url 2021-04-20-Comment-faire-une-api-web-en-python %}).

---

## 1. Installation de Flask-Caching

Installation de Flask-Caching, l’extension que nous utiliserons dans ce
tutoriel.

```bash
pip install Flask-Caching
```

---

## 2. Premier cache en mémoire avec SimpleCache

Objectif : mettre en place rapidement un cache en mémoire pour voir le
fonctionnement (init, cache de routes, memoize), tout en gardant à l’esprit ses
limites en production.

> Attention : SimpleCache stocke les données en mémoire dans le processus.
Avec Gunicorn (plusieurs workers), chaque worker a son propre cache.
Bien pour le dev, à éviter seul en prod.

### 2.1 Initialisation

```python
from flask import Flask
from flask_caching import Cache

app = Flask(__name__)
# Type "SimpleCache": en mémoire, non partagé entre processus
app.config["CACHE_TYPE"] = "SimpleCache"
app.config["CACHE_DEFAULT_TIMEOUT"] = 300  # 5 minutes
cache = Cache(app)
```

### 2.2 Cacher une route complète

```python
# Met en cache toute la vue pendant 60 secondes
@app.route("/time")
@cache.cached(timeout=60)
def server_time():
    import time
    return {"server_time": time.time()}
```

### 2.3 Cacher selon la query string

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

### 2.4 Mémoriser une fonction coûteuse (memoize)

```python
@cache.memoize(timeout=300)
def compute_stats(user_id: int):
    # Ici, faites des requêtes SQL lourdes, appels API, etc.
    import time
    time.sleep(2)
    return {"user_id": user_id, "score": 42}
```

### 2.5 Utiliser la fonction mémoïsée dans une route

```python
@app.route("/users/<int:user_id>/stats")
def user_stats(user_id: int):
    return compute_stats(user_id)
```

Points clés :

- `@cache.cached` met en cache la réponse d’une route (vue). Utilisez
  `query_string=True` si la réponse dépend des paramètres d’URL.
- `@cache.memoize` met en cache le résultat d’une fonction Python en fonction de
  ses arguments. Idéal pour encapsuler une requête coûteuse.

## 3. Clés personnalisées et cache par utilisateur

Dans certains cas, on veut construire des clés de cache spécifiques au contexte (utilisateur, paramètres) pour servir la bonne donnée à la bonne personne.

Pour des pages personnalisées, évitez d’utiliser le même cache pour tout le
monde. On peut ajouter un préfixe par utilisateur (id, rôle, etc.).

```python
from flask_login import current_user

@app.route("/dashboard")
@cache.cached(timeout=120, key_prefix=lambda: f"dashboard:{getattr(current_user, 'id', 'anon')}")
def dashboard():
    # Rendu personnalisé
    return {"hello": getattr(current_user, 'id', 'anon')}
```

> Astuce : pour des routes GET avec plusieurs critères, vous pouvez créer une
> clé
> unique basée sur la query string triée ou un hash.

```python
from flask import request
import hashlib

def qs_key(prefix: str = "view"):
    # Clé stable basée sur la query string (ex: view:abc123)
    raw = request.query_string or b""
    h = hashlib.sha1(raw).hexdigest()
    return f"{prefix}:{h}"

@app.route("/items")
@cache.cached(timeout=120, key_prefix=lambda: qs_key("items"))
def list_items():
    # ... charge la liste filtrée/paginée ...
    return {"items": [1, 2, 3]}
```

## 4. Invalidation

Quand et comment expirer/supprimer des entrées de cache ?

Trois niveaux :

- Vider une clé précise (niveau bas) :

```python
cache.delete("dashboard:123")
```

- Invalider une fonction mémorisée (memoize) :

```python
# Supprime tous les caches de compute_stats, tous arguments confondus
cache.delete_memoized(compute_stats)

# Ou seulement pour un argument précis
cache.delete_memoized(compute_stats, 123)
```

- Tout nettoyer  :

```python
cache.clear()
```

> Conseil : déclenchez l’invalidation après une écriture en base de données qui impacte la vue/fonction. Par exemple, après `POST /users/123`, invalidez `compute_stats(123)`.

---

## 6. Passer en production avec Redis

En production, on veut utiliser un backend partagé comme Redis pour que tous les workers
(et toutes les machines) partagent le même cache.

Installation :

```bash
# Redis côté serveur (exemple Ubuntu/Debian)
sudo apt-get install redis-server
# Client Python
pip install redis
```

Configuration Flask-Caching pour Redis :

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

> Avec des conteneurs/docker compose, référencez le service Redis via son nom de service
(`redis:6379`).

## 7. TTLs, tailles et formats

- Choisissez des TTL (timeouts) par type de données :
  - Métadonnées quasi statiques : 10–60 min
  - Listes paginées : 30–120 s
  - Détails utilisateurs : 60–300 s
- Sérialisation : Flask-Caching gère la sérialisation des fonctions. Cependant, si vous
  utilisez le cache en mode bas niveau, essayez de stocker du JSON :

```python
import json
cache.set("key", json.dumps({"x": 1}), timeout=60)
value = json.loads(cache.get("key") or "null")
```

## 8. Points d'attention

- SimpleCache + Gunicorn: le cache est dupliqué par worker → utilisez Redis en
  prod.
- Clés trop générales : vous servez la mauvaise donnée au mauvais utilisateur.
- Ne pas oublier `query_string=True` si la réponse dépend de la query string.
- Attention sur les invalidations trop agressive (clear global) → préférez `delete_memoized` ciblé.
- Cachez des opérations dépendantes d’un header (ex: langue). Dans ce cas,
  intégrez la langue dans la clé ou évitez le cache.

---

## 9. Exemple complet (Redis)

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

---

## Conclusion

Avec très peu de code, Flask-Caching apporte un gain de performance notable sur nos api.
Que ça soit avec `SimpleCache` pour le développement, ou Redis en
production pour partager le cache entre tous les workers.
N'oubliez pas de définir des TTL
adaptés, ainsi que de construire des clés intelligentes et adaptées au contexte (utilisateur, paramètres).

## Voir aussi

- [Flask-Caching](https://flask-caching.readthedocs.io/)
- [Redis](https://redis.io/)
- [Python : Comment faire des requêtes HTTP avec requests]({% post_url  2020-05-22-Comment-faire-des-requetes-http-en-python-avec-requests %})
- [Comment utiliser les sessions avec requests]({% post_url 2025-09-04-Comment-utiliser-les-sessions-avec-requests %})
- [Comment utiliser l'authentification avec requests]({% post_url 2025-09-05-Comment-utiliser-l-authentification-avec-requests %})
- [Comment faire une API web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %})
- [Organiser une application FastAPI en plusieurs fichiers]({% post_url 2025-08-17-Organiser-une-application-FastAPI-en-plusieurs-fichiers %})
