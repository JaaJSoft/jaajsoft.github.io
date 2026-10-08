---
layout: article
title: "Organiser une application FastAPI en plusieurs fichiers"
description: "Découper une application FastAPI en plusieurs fichiers avec APIRouter : un router par thème, un main.py qui les assemble, les __init__.py et le lancement."
author: Pierre Chopinet
tags:
  - python
  - fastapi
  - http
  - api
  - rest
  - architecture
  - routers
---

Tant qu'une API FastAPI tient en quelques routes, un seul fichier `app.py`
suffit. Quand elle grossit, mieux vaut ranger les routes par thème dans des
modules séparés. Dans ce tutoriel, nous allons découper une application avec
les `APIRouter` de FastAPI, et un point d'entrée qui les assemble.
<!--more-->

Dans cet article :
- Structure du projet
- Un router pour la route de statut
- Un router d'exemple
- Assembler l'application dans main.py
- Les fichiers `__init__.py`
- Lancer l'application
- Ajouter d'autres routers

Pré-requis : savoir créer une API minimale, voir
[Python : Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %}).
La route `/info/status` reprend celle de l'article
[Comment dockeriser une application FastAPI]({% post_url 2025-08-16-Comment-dockeriser-une-api-web-avec-FastAPI %}).

## Structure du projet

Voici l'organisation que nous allons mettre en place, qui convient bien à une
API de petite ou moyenne taille :

```
.
├── app
│   ├── __init__.py
│   ├── main.py              # Création de l'app et inclusion des routers
│   └── routers
│       ├── __init__.py
│       ├── exemple.py       # Router d'exemple
│       └── status.py        # Router dédié au statut/healthcheck
├── requirements.txt
└── README.md (optionnel)
```

Le fichier `app/main.py` crée l'instance FastAPI et y branche les routers.
Chaque fichier du dossier `app/routers` regroupe les routes d'un même thème :
`status.py` pour `/info/status`, `exemple.py` pour les routes `/exemple`.

## Un router pour la route de statut

Un `APIRouter` s'utilise comme l'objet `app` : on déclare les routes avec ses
décorateurs `get`, `post`, etc. Créez `app/routers/status.py` :

```python
from fastapi import APIRouter

router = APIRouter(prefix="/info", tags=["status"])  # préfixe commun à toutes les routes de ce router

@router.get("/status")
def info_status():
    # Réponse adaptée aux checks de disponibilité (k8s, Docker, etc.)
    return {"status": "ok"}
```

Le `prefix` est ajouté devant toutes les routes du router : la route déclarée
`/status` répond donc sur `/info/status`. Les `tags` servent à regrouper les
routes dans la documentation générée (`/docs`).

## Un router d'exemple

Même principe pour `app/routers/exemple.py` :

```python
from fastapi import APIRouter

router = APIRouter(prefix="/exemple", tags=["exemple"])  # préfixe commun à ce router

@router.get("/")
def list_exemples():
    return {"exemples": ["a", "b", "c"]}

@router.get("/{item_id}")
def get_exemple(item_id: int):
    return {"id": item_id, "nom": f"exemple {item_id}"}
```

Attention au `/` final : la première route répond sur `/exemple/`. Un appel sur
`/exemple` reçoit une redirection 307 vers `/exemple/`, que curl ne suit pas
sans l'option `-L`.

## Assembler l'application dans main.py

Reste à créer l'application et à y inclure les routers avec `include_router`.
Créez `app/main.py` :

```python
from fastapi import FastAPI
from .routers.status import router as status_router
from .routers.exemple import router as exemple_router


def create_app() -> FastAPI:
    app = FastAPI()

    # Exemple d'endpoint "racine" minimal
    @app.get("/")
    def root():
        return "Hello World"

    # On branche nos routers ici
    app.include_router(status_router)
    app.include_router(exemple_router)

    return app


# Uvicorn cherchera cette variable exportée
app = create_app()
```

Les imports relatifs (`from .routers.status ...`) désignent des modules du même
package `app`. Comme chaque module de router expose une variable nommée
`router`, on renomme ces variables à l'import pour pouvoir les utiliser côte à
côte. La fonction `create_app()` n'est pas obligatoire, mais elle permet de
fabriquer une application neuve à la demande, dans les tests par exemple.
Uvicorn, lui, utilise la variable `app` créée en bas du fichier.

## Les fichiers `__init__.py`

Créez enfin deux fichiers vides, `app/__init__.py` et `app/routers/__init__.py`.

Python 3 sait importer un dossier sans fichier `__init__.py` (il le traite
alors comme un *namespace package*), et l'application fonctionnerait sans eux.
Mais ces fichiers font de `app` et de `app/routers` des packages classiques :
c'est la convention, et c'est là qu'on placerait du code à exécuter à l'import
du package.

## Lancer l'application

Depuis la racine du projet (le dossier qui contient `app`) :

```bash
uvicorn app.main:app --reload
```

`app.main:app` désigne la variable `app` du module `app/main.py`. On peut
ensuite tester les routes :

```bash
curl http://127.0.0.1:8000/
"Hello World"

curl http://127.0.0.1:8000/info/status
{"status":"ok"}

curl http://127.0.0.1:8000/exemple/1
{"id":1,"nom":"exemple 1"}
```

La documentation reste disponible sur `http://127.0.0.1:8000/docs` (Swagger UI)
et `http://127.0.0.1:8000/redoc` (ReDoc), avec les routes groupées par tag.

Si vous avez dockerisé l'application comme dans l'article sur Docker, la route
`/info/status` ne change pas et la healthcheck continue de fonctionner. Par
contre, la commande de lancement du Dockerfile doit maintenant viser
`app.main:app` :

```dockerfile
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Avec l'ancien `app:app`, uvicorn importe le package `app` et s'arrête sur
l'erreur `Attribute "app" not found in module "app"`.

## Ajouter d'autres routers

Pour ajouter un nouveau groupe de routes, on crée un module dans
`app/routers`, par exemple `users.py` :

```python
from fastapi import APIRouter, Depends, Header, HTTPException


async def verifier_token(x_token: str = Header()):
    if x_token != "secret":
        raise HTTPException(status_code=401, detail="Token invalide")


router = APIRouter(
    prefix="/users",
    tags=["users"],
    dependencies=[Depends(verifier_token)],  # appliquée à toutes les routes du router
)

@router.get("/{user_id}")
def get_user(user_id: int):
    return {"id": user_id}
```

Puis on l'importe dans `main.py` et on l'inclut dans `create_app()` comme les
autres :

```python
from .routers.users import router as users_router

app.include_router(users_router)
```

Le paramètre `dependencies` de l'exemple montre un autre intérêt des routers :
une dépendance déclarée au niveau du router (ici la vérification d'un en-tête
`X-Token`) s'applique à toutes ses routes, sans avoir à la répéter sur chacune.
Un appel sur `/users/1` sans le bon en-tête est refusé :

```bash
curl -H "X-Token: faux" http://127.0.0.1:8000/users/1
{"detail":"Token invalide"}

curl -H "X-Token: secret" http://127.0.0.1:8000/users/1
{"id":1}
```

Par contre, les middlewares ne se déclarent pas sur un router : ils
s'appliquent à toute l'application, avec `app.add_middleware()`.

## Voir aussi

- [Python : Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %})
- [Comment dockeriser une application FastAPI]({% post_url 2025-08-16-Comment-dockeriser-une-api-web-avec-FastAPI %})
- [Uploader des fichiers avec FastAPI]({% post_url 2025-08-30-Comment-envoyer-des-fichiers-avec-FastAPI %})
- [Bigger Applications - Multiple Files](https://fastapi.tiangolo.com/tutorial/bigger-applications/), la documentation de FastAPI sur le sujet
