---
layout: article
title: "Ajouter un cache à notre application FastAPI avec redis"
description: "Ajouter un cache à une API FastAPI avec fastapi-cache2 : d'abord en mémoire, puis avec Redis pour le partager entre plusieurs processus."
author: Pierre Chopinet
tags:
- python
- fastapi
- http
- cache
- performance
- redis
- api
- rest

---

Quand une route fait un calcul coûteux ou interroge un service lent, garder sa
réponse quelques secondes évite de refaire le même travail à chaque requête
identique. Dans ce tutoriel, nous allons mettre en place ce cache avec la
librairie `fastapi-cache2`, d'abord en mémoire, puis dans Redis pour le partager
entre plusieurs processus. <!--more-->

Dans cet article :
- Installation
- Cache en mémoire
- Ce que le décorateur met en cache
- Cache Redis
- Personnaliser la clé de cache

Pré-requis : savoir démarrer une API minimale, voir
[Python : Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %}).

## Installation

On installe FastAPI, uvicorn, fastapi-cache2 et jinja2, puis le client Redis
pour la deuxième partie :

```bash
pip install fastapi uvicorn fastapi-cache2 jinja2
# Pour la partie Redis :
pip install redis
```

Sous Windows (PowerShell), vous pouvez faire :

```powershell
python -m pip install fastapi uvicorn fastapi-cache2 jinja2
python -m pip install redis
```

jinja2 n'est pas une dépendance déclarée de fastapi-cache2, mais il est devenu
nécessaire : la librairie importe le module de templates de Starlette (le
framework sur lequel repose FastAPI), qui refuse de se charger sans jinja2
depuis Starlette 1.0, sortie en mars 2026. Sans lui, l'import de `fastapi_cache`
échoue avec l'erreur `ImportError: jinja2 must be installed to use Jinja2Templates`.
Si vous avez installé FastAPI avec `pip install "fastapi[standard]"`, jinja2 est
déjà là.

Pour Redis, on installe le paquet `redis` à part, plutôt que l'extra
`fastapi-cache2[redis]` : celui-ci impose une version de redis antérieure à la
5, dont le client asynchrone n'a pas la méthode `aclose()` utilisée plus bas.

fastapi-cache2 n'a pas eu de nouvelle version depuis la 0.2.2, publiée en
juillet 2024. Les exemples de cet article ont été testés en octobre 2026 avec
fastapi-cache2 0.2.2, FastAPI 0.142.3, redis-py 8.1.0, Redis 7.0 et Python 3.13.

## Cache en mémoire

Le backend `InMemoryBackend` garde les réponses dans la mémoire du processus :
il n'y a rien d'autre à installer, c'est le plus simple pour commencer.

```python
# app_memory.py
import asyncio
from datetime import datetime, timezone
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
        "ts": datetime.now(timezone.utc).isoformat(timespec="seconds")
    }

# (Optionnel) vider le cache par namespace
@app.delete("/cache/clear")
async def clear_cache(namespace: str | None = None):
    await FastAPICache.clear(namespace=namespace)
    return {"cleared": True, "namespace": namespace}
```

`FastAPICache.init()` est appelé au démarrage de l'application, dans la fonction
`lifespan`. Le décorateur `@cache(expire=10)` garde ensuite la réponse de
`/slow` pendant 10 secondes. Pour simuler un traitement long, cette route attend
2 secondes avant de renvoyer l'heure courante.

On lance l'API :

```bash
uvicorn app_memory:app --reload
```

Le premier appel prend 2 secondes :

```bash
curl "http://127.0.0.1:8000/slow?q=42"
{"q":42,"ts":"2026-10-07T20:49:34+00:00"}
```

Le suivant, s'il arrive dans les 10 secondes, est immédiat et renvoie la même
heure. Avec `-i`, curl affiche aussi les en-têtes de la réponse :

```bash
curl -i "http://127.0.0.1:8000/slow?q=42"
HTTP/1.1 200 OK
date: Wed, 07 Oct 2026 20:49:34 GMT
server: uvicorn
content-length: 41
content-type: application/json
cache-control: max-age=9
etag: W/3506403070760895541
x-fastapi-cache: HIT

{"q":42,"ts":"2026-10-07T20:49:34+00:00"}
```

L'en-tête `x-fastapi-cache` vaut `MISS` quand la réponse vient d'être calculée
et `HIT` quand elle sort du cache. fastapi-cache2 ajoute aussi un
`cache-control` avec la durée de vie restante, en secondes, et un `ETag`.

Ce cache a ses limites : il vit dans la mémoire du processus, il est donc perdu
à chaque redémarrage, et il n'est pas partagé. Avec `uvicorn --workers 4`,
chaque worker a son propre cache. De plus, `InMemoryBackend` ne supprime une
entrée expirée que lorsqu'on la relit : une route appelée avec beaucoup de
valeurs différentes (une entrée par valeur de `q` ici) fait grossir la mémoire
du processus.

## Ce que le décorateur met en cache

L'ordre des décorateurs compte : `@cache` doit se trouver sous `@app.get`, pour
envelopper la fonction avant que FastAPI ne l'enregistre. Dans l'autre sens, la
route fonctionne mais rien n'est mis en cache.

La clé de cache est calculée à partir du module et du nom de la fonction, et des
valeurs de ses arguments : `/slow?q=42` et `/slow?q=43` donnent deux entrées
différentes. Ce qui n'est pas un argument de la fonction n'entre pas dans la
clé : ni les en-têtes de la requête, ni un paramètre d'URL que la fonction ne
déclare pas (`/slow?q=42&foo=1` renvoie la réponse en cache de `/slow?q=42`).

Seules les requêtes GET sont mises en cache. Et le client peut contourner le
cache : avec l'en-tête `Cache-Control: no-store`, la réponse est recalculée sans
passer par le cache, et avec `Cache-Control: no-cache`, elle est recalculée puis
remise en cache. N'importe qui peut donc forcer le recalcul d'une route : le
cache accélère les réponses, il ne protège pas une route coûteuse contre les
abus (c'est le rôle d'un
[rate limiter]({% post_url 2025-09-20-Limiter-le-rate-d-une-API-FastAPI-avec-Redis %})).

Enfin, la route `DELETE /cache/clear` vide le cache : sans namespace,
`FastAPICache.clear()` supprime toutes les clés de l'application. En production,
une route comme celle-ci doit être protégée par une authentification, sinon
n'importe qui peut vider votre cache.

## Cache Redis

Avec Redis, le cache vit en dehors de l'application : il est partagé entre tous
les workers et toutes les instances de l'API, et il survit à leurs redémarrages.

### Démarrer un Redis local (docker-compose)

Le plus simple pour avoir un Redis en local est de passer par Docker, avec ce
fichier `docker-compose.yml` :

```yaml
# docker-compose.yml
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: [ "redis-server", "--appendonly", "yes" ]
```

Lancez-le :

```bash
docker compose up -d
```

L'option `--appendonly yes` active la persistance de Redis sur disque : le cache
survit aussi à un redémarrage de Redis.

### Le code avec RedisBackend

```python
# app_redis.py
import asyncio
from datetime import datetime, timezone
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
    await redis_client.aclose()

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
        "ts": datetime.now(timezone.utc).isoformat(timespec="seconds")
    }

@app.delete("/cache/clear")
async def clear_cache(namespace: str | None = None):
    await FastAPICache.clear(namespace=namespace)
    return {"cleared": True, "namespace": namespace}
```

Par rapport à la version en mémoire, on crée un client Redis asynchrone au
démarrage, et on le referme à l'arrêt de l'application. Ce client doit garder
`decode_responses=False` (la valeur par défaut) : fastapi-cache2 stocke des
bytes, et un client qui décode les réponses en texte casse le cache. La route
`/slow` est rangée dans le namespace `v1`, ce qui permet de vider d'un coup
toutes les clés de ce groupe.

On lance l'API :

```bash
uvicorn app_redis:app --reload
```

Le premier appel prend 2 secondes, le deuxième est immédiat. Après la purge du
namespace `v1`, la réponse est recalculée, avec une nouvelle heure :

```bash
curl "http://127.0.0.1:8000/slow?q=1"
{"q":1,"ts":"2026-10-07T20:49:55+00:00"}
curl "http://127.0.0.1:8000/slow?q=1"
{"q":1,"ts":"2026-10-07T20:49:55+00:00"}
curl -X DELETE "http://127.0.0.1:8000/cache/clear?namespace=v1"
{"cleared":true,"namespace":"v1"}
curl "http://127.0.0.1:8000/slow?q=1"
{"q":1,"ts":"2026-10-07T20:49:57+00:00"}
```

L'avantage de Redis, c'est qu'on voit ce qu'il contient :

```bash
docker compose exec redis redis-cli keys 'fastapi-cache:*'
fastapi-cache:v1:1dfdffccea5a80189660c039da617fcb
```

La clé est formée du préfixe passé à `FastAPICache.init()`, du namespace, puis
d'un hash MD5 du module, du nom de la fonction et de ses arguments. Avec Redis,
choisissez un préfixe explicite (le nom de votre application par exemple) pour
ne pas mélanger vos clés avec celles d'une autre application qui utiliserait la
même base.

Attention, `FastAPICache.clear()` s'appuie sur la commande `KEYS` de Redis, qui
parcourt toutes les clés de la base : la documentation de Redis déconseille de
l'utiliser en production sur une grosse base, car elle peut dégrader fortement
les performances.

## Personnaliser la clé de cache

Par défaut, deux utilisateurs qui appellent la même route avec les mêmes
paramètres reçoivent la même réponse en cache. Si la réponse dépend d'un
en-tête, comme la langue ou l'utilisateur, il faut l'ajouter à la clé avec un
`key_builder`.

Attention à la signature : dans fastapi-cache2 0.2, `default_key_builder` attend
`request`, `response`, `args` et `kwargs` en arguments nommés uniquement (placés
après le `*`). Le builder personnalisé reprend donc la même signature et
transmet ces valeurs par nom. Ajoutez dans `app_redis.py` :

```python
from fastapi import Request, Response
from fastapi_cache.key_builder import default_key_builder

def user_lang_key_builder(
    func,
    namespace: str = "",
    *,
    request: Request = None,
    response: Response = None,
    args: tuple = (),
    kwargs: dict = None,
) -> str:
    # Repart d'un builder par défaut et y ajoute l'Accept-Language et un user-id (fictif)
    base = default_key_builder(
        func,
        namespace,
        request=request,
        response=response,
        args=args,
        kwargs=kwargs or {},
    )
    lang = request.headers.get("accept-language", "*") if request else "*"
    user = request.headers.get("x-user-id", "anon") if request else "anon"
    return f"{base}:u={user}:lang={lang}"

@app.get("/profil")
@cache(expire=60, namespace="v1", key_builder=user_lang_key_builder)
async def profil():
    return {"ts": datetime.now(timezone.utc).isoformat(timespec="seconds")}
```

Chaque combinaison utilisateur et langue a maintenant sa propre entrée :

```bash
curl -H "X-User-Id: 42" -H "Accept-Language: fr" http://127.0.0.1:8000/profil
curl -H "X-User-Id: 7" -H "Accept-Language: en" http://127.0.0.1:8000/profil
curl http://127.0.0.1:8000/profil
```

Une fois la clé de `/slow` expirée, Redis contient :

```bash
docker compose exec redis redis-cli keys 'fastapi-cache:*'
fastapi-cache:v1:2420ee266db3d4bd2cf91a64838cb258:u=42:lang=fr
fastapi-cache:v1:2420ee266db3d4bd2cf91a64838cb258:u=7:lang=en
fastapi-cache:v1:2420ee266db3d4bd2cf91a64838cb258:u=anon:lang=*
```

Ici, l'identifiant de l'utilisateur vient d'un en-tête `X-User-Id` pour
l'exemple. Dans une vraie application, il doit venir de l'authentification :
sinon, il suffit de changer l'en-tête pour lire le cache d'un autre utilisateur.

## Voir aussi

- [Python : Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %})
- [Comment ajouter un rate limiter à notre application FastAPI avec redis]({% post_url 2025-09-20-Limiter-le-rate-d-une-API-FastAPI-avec-Redis %})
- [Comment ajouter un cache à une application Flask]({% post_url 2025-09-14-Comment-utiliser-un-cache-avec-Flask %})
- [Python : Mettre en cache des fonctions avec lru_cache]({% post_url 2026-05-25-Python-lru_cache %})
- [fastapi-cache sur GitHub](https://github.com/long2ice/fastapi-cache)
- [fastapi-cache2 sur PyPI](https://pypi.org/project/fastapi-cache2/)
