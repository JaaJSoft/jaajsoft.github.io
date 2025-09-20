---
layout: article
title: "Comment ajouter un rate limiter à notre application FastAPI avec redis"
author: Pierre Chopinet
tags:
  - python
  - fastapi
  - redis
  - rate-limit
  - securité
  - performance
  - tutoriel
---

Dans ce tutoriel, on met en place un rate limiting pour une API FastAPI à l’aide
de
la bibliothèque `fastapi-limiter`, avec Redis.
<!--more-->
L’objectif est de protéger vos
endpoints contre les abus (pics de trafic, scripts, DDoS applicatif léger) et de
mieux contrôler votre consommation de ressources.

Prérequis : savoir démarrer une API minimaliste.
Voir [Python : Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %}).

## Installation

Installez les dépendances nécessaires :

```bash
pip install fastapi uvicorn fastapi-limiter redis
```

Sous Windows (PowerShell), vous pouvez faire:

```powershell
python -m pip install fastapi uvicorn fastapi-limiter redis
```

## Démarrer un Redis local (docker-compose)

On utilise docker pour déployer un serveur Redis rapidement en local :

```yaml
# docker-compose.yml
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: [ "redis-server", "--appendonly", "yes" ]
```

Lancez:

```bash
docker compose up -d
```

---

## Mise en place minimale avec fastapi-limiter

`fastapi-limiter` s’initialise au démarrage de l’application avec un client
Redis.
Ensuite, on ajoute une dépendance `RateLimiter` sur les routes à protéger.

### Mise en place du limiter

On initialise le rater limiter dans la méthode lifespan utilisé par FastAPI :

```python
# app_rate_limit_min.py
from contextlib import asynccontextmanager
from fastapi import FastAPI, Depends
from fastapi_limiter import FastAPILimiter
from fastapi_limiter.depends import RateLimiter
import redis.asyncio as redis

redis_client: redis.Redis | None = None

@asynccontextmanager
async def lifespan(app: FastAPI):
    global redis_client
    # Connexion Redis (adapter l'URL si besoin: auth, DB, TLS, etc.)
    redis_client = redis.from_url(
        "redis://localhost:6379", encoding="utf-8"
    )
    await FastAPILimiter.init(redis_client, prefix="fastapi-limiter")
    yield
    # Fermeture propre
    assert redis_client is not None
    await redis_client.close()

app = FastAPI(lifespan=lifespan)
```

> Note : Vous pouvez réutiliser le même client redis que pour votre cache

Puis, on ajoute à nos routes notre limiteur :

```python
# Cette route autorise 5 requêtes par minute (par identifiant; voir plus bas)
@app.get("/ping", dependencies=[Depends(RateLimiter(times=5, seconds=60))])
async def ping():
    return {"pong": True}
```

Démarrage :

```bash
uvicorn app:app --reload
```

Test rapide :

```bash
# Faites plus de 5 appels en moins de 60s pour observer l'erreur 429
for i in {1..7}; do curl -i http://127.0.0.1:8000/ping; echo; done
```

Par défaut, `fastapi-limiter` identifie le client par son IP (en s’aidant des
en-têtes classiques si vous utilisez un proxy). Vous pouvez cependant personnaliser cet
identifiant si besoin.

---

## Personnaliser l’identifiant (IP, clé API, utilisateur, etc.)

Souvent, on veut limiter par clé API ou par utilisateur authentifié plutôt que
par IP.
Pour cela, on peut fournir une fonction `identifier` au `RateLimiter`.

```python
from fastapi import Request

# Limite 100 requêtes par 24h et par clé API (X-API-Key), sinon par IP
@app.get(
    "/data",
    dependencies=[
        Depends(
            RateLimiter(
                times=100,
                hours=24,
                identifier=lambda request: request.headers.get("X-API-Key") or request.client.host,
            )
        )
    ],
)
async def get_data(request: Request):
    return {"ok": True}
```

Autre exemple : limitation par utilisateur connecté (ex: `request.state.user.id`
ou
`request.user.id` selon votre middleware d’authentification).

```python
@app.get(
    "/me",
    dependencies=[
        Depends(
            RateLimiter(
                times=60,
                minutes=1,
                identifier=lambda req: getattr(getattr(req, "user", None), "id", None) or req.client.host,
            )
        )
    ],
)
async def me():
    return {"me": True}
```

---

## Appliquer une limite par défaut à un groupe de routes

Vous pouvez appliquer un rate limit à l’échelle d’un router, afin qu’il
s’applique
à toutes les routes incluses.

```python
from fastapi import APIRouter

api_router = APIRouter(
    prefix="/api",
    dependencies=[Depends(RateLimiter(times=120, minutes=1))],  # par défaut: 120 req/min
)

@api_router.get("/items")
async def list_items():
    return {"items": []}

@api_router.post("/items")
async def create_item():
    return {"created": True}

app.include_router(api_router)
```

Vous pouvez toujours surcharger/compléter le comportement sur une route précise
en
ajoutant un autre `Depends(RateLimiter(...))` directement sur l’endpoint.

---

## Cas derrière un reverse proxy (Nginx, Traefik, Cloudflare)

Pour que l’IP réelle du client soit correctement vue, pensez à activer la prise
en
compte des en‑têtes proxy. Par exemple:

```python
from starlette.middleware import Middleware
from starlette.middleware.proxy_headers import ProxyHeadersMiddleware

# Au besoin, passez la liste des proxies de confiance (num_trusted_hops)
app = FastAPI(lifespan=lifespan, middleware=[Middleware(ProxyHeadersMiddleware, trusted_hosts="*")])
```

---

## Gérer le 429 Too Many Requests

Quand la limite est dépassée, `fastapi-limiter` soulève une
`HTTPException(429)`.
Vous pouvez personnaliser la gestion globale avec un handler FastAPI pour
retourner
un message JSON cohérent avec votre API.

```python
from fastapi import Request, HTTPException
from fastapi.responses import JSONResponse

@app.exception_handler(429)
async def too_many_requests_handler(request: Request, exc: HTTPException):
    return JSONResponse(
        status_code=429,
        content={
            "error": "too_many_requests",
            "detail": exc.detail or "Rate limit exceeded",
        },
        headers={"Retry-After": "60"},
    )
```

---

## Bonnes pratiques et points d’attention

- Granularité : adaptez les paramètres (secondes, minutes, heures) à vos usages.
- Identifiant : préférez un identifiant stable (clé API, user id) quand c’est
  pertinent.
- Proxies : gérez correctement les IP réelles (ProxyHeadersMiddleware, trusted
  hops).
- Endpoints sensibles : combinez avec de l’authentification, voire du captcha sur
  les routes publiques.
- Observabilité : loggez les 429 et surveillez vos métriques.

---

## Pour aller plus loin

- GitHub: [long2ice/fastapi-limiter](https://github.com/long2ice/fastapi-limiter)
- [Ajouter un cache à notre application FastAPI avec redis]({% post_url 2025-08-18-Utiliser-fastapi-cache2-avec-FastAPI %})
- [Python : Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %})
- [Organiser une application FastAPI en plusieurs fichiers]({% post_url 2025-08-17-Organiser-une-application-FastAPI-en-plusieurs-fichiers %})

