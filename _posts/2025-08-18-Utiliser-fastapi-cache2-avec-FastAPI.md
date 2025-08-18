---
layout: article
title: "Ajouter un cache à notre application FastAPI avec redis"
author: Pierre Chopinet
tags:
- python
- fastapi
- cache
- performance
- redis
- tutoriel

---

Dans ce tutoriel, on met en place un cache d'API avec la librairie
`fastapi-cache2`.
On commence par un cache en mémoire (simple, sans dépendance), puis on passe à
Redis pour
un cache partagé et persistant. <!--more-->

Prérequis: savoir démarrer une API minimaliste. Si besoin, lisez d’abord
[Python : Comment faire une api web avec FastAPI]({% post_url
2025-08-15-Comment-faire-une-api-web-avec-FastAPI %}).

## Installation

Installez les dépendances nécessaires:

```bash
pip install fastapi uvicorn fastapi-cache2
# Pour la partie Redis:
pip install redis
```

Sous Windows (PowerShell), vous pouvez faire:

```powershell
python -m pip install fastapi uvicorn fastapi-cache2
python -m pip install redis
```

## Rappel: pourquoi mettre du cache ?

- Réduire la charge CPU/IO lorsque les mêmes requêtes reviennent souvent.
- Accélérer les réponses (moins d’appels vers des services externes ou bases de
  données).
- Stabiliser la latence pour certaines routes "coûteuses".

`fastapi-cache2` propose un décorateur `@cache()` qui mémorise le résultat d’une
route
pendant une durée donnée. Vous pouvez choisir le backend: mémoire (
InMemoryBackend) ou Redis.

---

## Partie 1 — Cache en mémoire

Avantages: simple, aucune dépendance externe. Limites: non partagé entre
plusieurs processus/
conteneurs; vidé à chaque redémarrage.

### Code complet (InMemoryBackend)

```python
# app_memory.py
import asyncio
from datetime import datetime
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi_cache import FastAPICache
from fastapi_cache.backends.inmemory import InMemoryBackend
from fastapi_cache.decorator import cache

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Initialisation du cache côté application (au démarrage)
    FastAPICache.init(InMemoryBackend(), prefix="fastapi-cache")
    yield
    # Rien à nettoyer en fin de vie pour l'in-memory

app = FastAPI(lifespan=lifespan)

@app.get("/")
async def root():
    return {"service": "demo-cache", "backend": "memory"}

@app.get("/slow")
@cache(expire=10)  # la réponse est mise en cache 10 secondes
async def slow_endpoint(q: int = 1):
    # Simule un travail coûteux
    await asyncio.sleep(2)
    return {
        "q": q,
        "ts": datetime.utcnow().isoformat(timespec="seconds")
    }

# (Optionnel) vider le cache par namespace
@app.delete("/cache/clear")
async def clear_cache(namespace: str | None = None):
    await FastAPICache.clear(namespace=namespace)
    return {"cleared": True, "namespace": namespace}
```

Démarrage:

```bash
uvicorn app_memory:app --reload
```

Test rapide (notez le timestamp qui ne change pas pendant 10s):

```bash
# 1ère requête lente (~2s) puis réponse mise en cache
curl "http://127.0.0.1:8000/slow?q=42"
# 2ème requête immédiate (<10s), même payload (ts identique)
curl "http://127.0.0.1:8000/slow?q=42"
```

Notes importantes:

- `@cache(expire=10)` définit la durée de vie (TTL) pour cette route.
- La clé de cache par défaut inclut l’URL et les paramètres de requête. Les
  en-têtes ne sont
  pas pris en compte par défaut.
- Utilisez `namespace="v1"` dans le décorateur pour regrouper des caches et les
  purger d’un coup
  via `FastAPICache.clear(namespace="v1")`.

---

## Partie 2 — Cache Redis

Avantages: partagé entre plusieurs workers/instances, persistance (selon
config),
observabilité (on voit les clés). Nécessite un service Redis.

### Démarrer un Redis local (docker-compose)

```yaml
# docker-compose.yml
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: [ "redis-server", "--appendonly", "yes" ]
```

Lancez-le:

```bash
docker compose up -d
```

### Code complet (RedisBackend)

```python
# app_redis.py
import asyncio
from datetime import datetime
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi_cache import FastAPICache
from fastapi_cache.backends.redis import RedisBackend
from fastapi_cache.decorator import cache
import redis.asyncio as redis

redis_client: redis.Redis | None = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    global redis_client
    # Connexion Redis (adapter l'URL si besoin, auth, DB, etc.)
    redis_client = redis.from_url(
        "redis://localhost:6379", encoding="utf-8", decode_responses=False
    )
    FastAPICache.init(RedisBackend(redis_client), prefix="fastapi-cache")
    yield
    # Fermeture propre de la connexion Redis
    assert redis_client is not None
    await redis_client.close()

app = FastAPI(lifespan=lifespan)

@app.get("/")
async def root():
    return {"service": "demo-cache", "backend": "redis"}

@app.get("/slow")
@cache(expire=15, namespace="v1")  # 15s de cache, dans le namespace v1
async def slow_endpoint(q: int = 1):
    await asyncio.sleep(2)
    return {
        "q": q,
        "ts": datetime.utcnow().isoformat(timespec="seconds")
    }

@app.delete("/cache/clear")
async def clear_cache(namespace: str | None = None):
    await FastAPICache.clear(namespace=namespace)
    return {"cleared": True, "namespace": namespace}
```

Démarrage:

```bash
uvicorn app_redis:app --reload
```

Vérification du cache:

```bash
curl "http://127.0.0.1:8000/slow?q=1"   # lente
curl "http://127.0.0.1:8000/slow?q=1"   # immédiate (<15s), même payload
curl -X DELETE "http://127.0.0.1:8000/cache/clear?namespace=v1"
curl "http://127.0.0.1:8000/slow?q=1"   # lente à nouveau (cache vidé)
```

### key_builder personnalisé (optionnel)

Dans certains cas, on veut pouvoir customiser notre clé de cache.
FastAPI propose un key_builder qui permet de faire cela.

Exemple ci-dessous :

```python
from fastapi import Request
from fastapi_cache import JsonCoder
from fastapi_cache.key_builder import default_key_builder

async def user_lang_key_builder(func, namespace: str, request: Request, response=None, *args, **kwargs) -> str:
    # Repart d’un builder par défaut et y ajoute l’Accept-Language et un user-id (fictif)
    base = default_key_builder(func, namespace, request, response, *args, **kwargs)
    lang = request.headers.get("accept-language", "*")
    user = request.headers.get("x-user-id", "anon")
    return f"{base}:u={user}:lang={lang}"

# À l'usage:
# @cache(expire=60, namespace="v1", key_builder=user_lang_key_builder, coder=JsonCoder)
```

### Bonnes pratiques et points d’attention

- Préfixe: définissez un `prefix` explicite (par exemple avec le nom de votre
  app et la version) pour isoler vos clés.
- Namespace: utile pour invalider sélectivement des sous-ensembles de clés.
- Sécurité: éviter d’exposer un endpoint de purge sans protection; ajoutez
  auth/rôle.
- TTL (time to live): choisissez une durée adaptée à la fraîcheur des données et
  au coût de recalcul.
- Multi‑workers: avec Uvicorn/Gunicorn en multi‑processus, utilisez Redis (
  l’in‑memory n’est pas partagé entre workers).

---

## Pour aller plus loin

- Documentation fastapi-cache2
  - GitHub: https://github.com/fastapi-cache/fastapi-cache
  - PyPI: https://pypi.org/project/fastapi-cache2/

- Autres articles FastAPI sur ce blog
  - [Python : Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %})
  - [Comment dockeriser une API FastAPI]({% post_url 2025-08-16-Comment-dockeriser-une-api-web-avec-FastAPI %})
  - [Organiser une application FastAPI en plusieurs fichiers]({% post_url 2025-08-17-Organiser-une-application-FastAPI-en-plusieurs-fichiers %})
