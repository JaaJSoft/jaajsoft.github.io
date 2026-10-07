---
layout: article
title: "Comment ajouter un rate limiter à notre application FastAPI avec redis"
author: Pierre Chopinet
tags:
  - python
  - fastapi
  - http
  - api
  - rest
  - redis
  - rate-limit
  - sécurité
  - performance
---

> **Note (2026) :** fastapi-limiter a changé depuis l'écriture de cet article. La version 0.2.0 (février 2026) abandonne l'API utilisée ici. Et depuis FastAPI 0.137 (juin 2026), fastapi-limiter, en 0.1.6 comme en 0.2.0, provoque une erreur 500 sur les routes limitées dès que l'application utilise `include_router`. Les exemples restent valables avec les versions épinglées dans la partie installation : ils ont été testés en octobre 2026 avec FastAPI 0.136.3, fastapi-limiter 0.1.6 et redis-py 8.1.0.

Dans ce tutoriel, nous allons limiter le nombre de requêtes qu'un client peut
faire sur une API FastAPI, avec la librairie `fastapi-limiter` et Redis.
Au-delà de la limite, l'API répond `429 Too Many Requests` : de quoi se protéger
d'un script trop gourmand, d'un client qui boucle, ou limiter la consommation
d'une route coûteuse.
<!--more-->

Dans cet article :
- Installation
- Démarrer un Redis local
- Mise en place minimale
- Limiter par clé API ou par utilisateur
- Limiter un groupe de routes
- Derrière un reverse proxy
- Personnaliser la réponse 429

Pré-requis : savoir démarrer une API minimale, voir
[Python : Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %}).

## Installation

```bash
pip install "fastapi<0.137" uvicorn "fastapi-limiter==0.1.6" redis
```

Sous Windows (PowerShell), vous pouvez faire :

```powershell
python -m pip install "fastapi<0.137" uvicorn "fastapi-limiter==0.1.6" redis
```

Les deux versions sont épinglées. fastapi-limiter 0.1.6 est la dernière version
construite directement sur Redis : la 0.2.0 confie le comptage à la librairie
pyrate-limiter (qui peut elle-même stocker ses compteurs dans Redis), avec une
autre API. `FastAPILimiter.init()` comme les paramètres `times` et `seconds`
utilisés plus bas n'y existent plus.

Quant à FastAPI, la version 0.137 a changé le contenu de `app.routes` : on y
trouve maintenant des objets intermédiaires pour les routers inclus, et plus
seulement des routes. Or fastapi-limiter parcourt cette liste à chaque requête
pour retrouver la route appelée. Dès que l'application contient un
`include_router`, chaque requête sur une route limitée échoue alors avec
`AttributeError: '_IncludedRouter' object has no attribute 'path'`, et
l'utilisateur reçoit une erreur 500. La 0.136.3 est la dernière version de
FastAPI avec laquelle les exemples de cet article fonctionnent.

## Démarrer un Redis local

On utilise Docker pour lancer un serveur Redis en local, avec ce fichier
`docker-compose.yml` :

```yaml
# docker-compose.yml
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: [ "redis-server", "--appendonly", "yes" ]
```

Lancez :

```bash
docker compose up -d
```

## Mise en place minimale

`fastapi-limiter` s'initialise au démarrage de l'application, avec un client
Redis, dans la fonction `lifespan` utilisée par FastAPI :

```python
# app.py
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
        "redis://localhost:6379", encoding="utf-8", decode_responses=True
    )
    await FastAPILimiter.init(redis_client, prefix="fastapi-limiter")
    yield
    # Fermeture propre
    assert redis_client is not None
    await redis_client.aclose()

app = FastAPI(lifespan=lifespan)
```

Si votre application utilise déjà Redis pour son cache, comme dans l'article
[Ajouter un cache à notre application FastAPI avec redis]({% post_url 2025-08-18-Utiliser-fastapi-cache2-avec-FastAPI %}),
vous pouvez réutiliser le même client. Il doit alors être créé avec
`decode_responses=False`, comme l'exige fastapi-cache2 : fastapi-limiter
fonctionne aussi dans ce mode.

Ensuite, on protège une route en lui ajoutant la dépendance `RateLimiter` :

```python
# Cette route autorise 5 requêtes par minute (par identifiant; voir plus bas)
@app.get("/ping", dependencies=[Depends(RateLimiter(times=5, seconds=60))])
async def ping():
    return {"pong": True}
```

On lance l'API :

```bash
uvicorn app:app --reload
```

Puis on l'appelle 7 fois de suite, en n'affichant que le code HTTP de la
réponse :

```bash
for i in {1..7}; do curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8000/ping; done
200
200
200
200
200
429
429
```

Les 5 premiers appels passent, les suivants sont refusés jusqu'à la fin de la
minute. La réponse indique combien de secondes attendre, dans l'en-tête
`retry-after` :

```bash
curl -i http://127.0.0.1:8000/ping
HTTP/1.1 429 Too Many Requests
date: Wed, 07 Oct 2026 20:54:26 GMT
server: uvicorn
retry-after: 60
content-length: 30
content-type: application/json

{"detail":"Too Many Requests"}
```

fastapi-limiter compte les requêtes dans Redis avec une fenêtre fixe : le
premier appel crée un compteur qui expire au bout de 60 secondes, les suivants
l'incrémentent, et une fois la limite atteinte, les requêtes sont refusées
jusqu'à l'expiration du compteur. On voit ce compteur dans Redis :

```bash
docker compose exec redis redis-cli keys 'fastapi-limiter:*'
fastapi-limiter:127.0.0.1:/ping:4:0
```

La clé contient l'identifiant du client, par défaut son IP suivie du chemin
appelé, puis la position de la route dans l'application : chaque route a son
propre compteur.

Attention, pour trouver l'IP du client, l'identifiant par défaut lit d'abord
l'en-tête `X-Forwarded-For`, sans vérifier d'où vient la requête. Un client qui
envoie une valeur différente de cet en-tête à chaque appel obtient un nouveau
compteur à chaque fois, et n'est donc jamais bloqué. La partie sur le reverse
proxy montre comment corriger ça.

## Limiter par clé API ou par utilisateur

On peut préférer limiter par clé API ou par utilisateur plutôt que par IP. Pour
cela, on passe une fonction `identifier` au `RateLimiter`. fastapi-limiter
l'appelle avec `await` : il doit donc s'agir d'une fonction asynchrone
(`async def`), et non d'un `lambda` synchrone.

```python
from fastapi import Request

# Limite par clé API (X-API-Key) si présente, sinon par IP
async def api_key_identifier(request: Request) -> str:
    return request.headers.get("X-API-Key") or request.client.host

# Limite 100 requêtes par 24h et par clé API (X-API-Key), sinon par IP
@app.get(
    "/data",
    dependencies=[
        Depends(
            RateLimiter(
                times=100,
                hours=24,
                identifier=api_key_identifier,
            )
        )
    ],
)
async def get_data(request: Request):
    return {"ok": True}
```

Pour limiter par utilisateur connecté, l'identifiant dépend de votre
authentification. Avec un `AuthenticationMiddleware` de Starlette, l'utilisateur
est rangé dans `request.scope["user"]` :

```python
async def user_identifier(request: Request) -> str:
    # Utilisateur posé par le middleware d'authentification s'il y en a un, sinon IP
    user = request.scope.get("user")
    return str(getattr(user, "id", None) or request.client.host)

@app.get(
    "/me",
    dependencies=[
        Depends(
            RateLimiter(
                times=60,
                minutes=1,
                identifier=user_identifier,
            )
        )
    ],
)
async def me():
    return {"me": True}
```

On passe par `request.scope` plutôt que par `request.user` : sans middleware
d'authentification, `request.user` lève une `AssertionError`, que
`getattr(request, "user", None)` ne rattrape pas (il ne gère que les
`AttributeError`). La route répondrait alors par une erreur 500.

## Limiter un groupe de routes

On peut aussi déclarer la limite sur un router, pour qu'elle s'applique à toutes
ses routes :

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

Attention, la dépendance est ajoutée à chaque route : `GET /api/items` et
`POST /api/items` ont chacune leur compteur de 120 requêtes par minute, ce n'est
pas un quota commun à tout le groupe.

Un `Depends(RateLimiter(...))` ajouté sur une des routes du router impose une
seconde limite, qui s'applique en plus de la première : c'est la plus stricte
qui bloque. On peut donc rendre une route plus restrictive que le reste du
groupe, mais pas plus permissive.

C'est ce `include_router` qui provoque les erreurs 500 avec FastAPI 0.137 et
suivants, d'où la version de FastAPI épinglée à l'installation.

## Derrière un reverse proxy

Derrière un reverse proxy (Nginx, Traefik...), toutes les requêtes arrivent de
l'IP du proxy, et c'est l'en-tête `X-Forwarded-For` qui porte l'IP du client.
Uvicorn sait lire cet en-tête, mais par défaut il ne lui fait confiance que pour
les requêtes qui viennent de `127.0.0.1` ou `::1`, c'est-à-dire d'un proxy
installé sur la même machine. Si le proxy est ailleurs (dans un autre conteneur par exemple),
on donne son adresse avec l'option `--forwarded-allow-ips` :

```bash
uvicorn app:app --host 0.0.0.0 --forwarded-allow-ips 10.0.0.5
```

`request.client.host` contient alors l'IP du client, et non celle du proxy. La
valeur `"*"` fait confiance à toutes les adresses : à réserver aux cas où
personne ne peut joindre l'application sans passer par le proxy.

Comme l'identifiant par défaut de fastapi-limiter lit `X-Forwarded-For` sans
tenir compte de ce réglage, mieux vaut le remplacer, pour toute l'application,
par un identifiant basé sur `request.client.host` :

```python
from fastapi import Request

async def ip_identifier(request: Request) -> str:
    # request.client.host : IP corrigée par uvicorn, uniquement si la requête
    # vient d'un proxy de confiance (--forwarded-allow-ips)
    return request.client.host + ":" + request.scope["path"]

# Dans lifespan :
await FastAPILimiter.init(redis_client, prefix="fastapi-limiter", identifier=ip_identifier)
```

Un client qui envoie directement à l'API un faux `X-Forwarded-For` reste alors
compté sous sa vraie IP.

## Personnaliser la réponse 429

Quand la limite est dépassée, `fastapi-limiter` lève une `HTTPException` 429,
avec l'en-tête `Retry-After`. Pour renvoyer un JSON cohérent avec le reste de
votre API, on peut déclarer un gestionnaire d'exception :

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
        headers=exc.headers,  # contient déjà le Retry-After calculé par fastapi-limiter
    )
```

```bash
curl -i http://127.0.0.1:8000/ping
HTTP/1.1 429 Too Many Requests
date: Wed, 07 Oct 2026 20:54:46 GMT
server: uvicorn
retry-after: 56
content-length: 58
content-type: application/json

{"error":"too_many_requests","detail":"Too Many Requests"}
```

On reprend les en-têtes de l'exception, qui contiennent le nombre exact de
secondes à attendre : une valeur fixe comme `"60"` serait fausse pour la limite
de 24 heures de la route `/data`.

## Voir aussi

- [Python : Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %})
- [Ajouter un cache à notre application FastAPI avec redis]({% post_url 2025-08-18-Utiliser-fastapi-cache2-avec-FastAPI %})
- [Organiser une application FastAPI en plusieurs fichiers]({% post_url 2025-08-17-Organiser-une-application-FastAPI-en-plusieurs-fichiers %})
- [fastapi-limiter sur GitHub](https://github.com/long2ice/fastapi-limiter)
- [Les notes de version de FastAPI](https://fastapi.tiangolo.com/release-notes/), voir la 0.137.0 pour le changement sur les routers
