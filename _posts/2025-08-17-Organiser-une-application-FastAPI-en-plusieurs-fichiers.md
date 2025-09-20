---
layout: article
title: "Organiser une application FastAPI en plusieurs fichiers"
author: Pierre Chopinet
tags:
  - python
  - fastapi
  - api
  - tutoriel
  - architecture
  - routers
---

Dans ce tutoriel, vous allez apprendre à structurer proprement une application
FastAPI en plusieurs fichiers et modules Python, en séparant vos routes, votre
configuration et votre point d'entrée.
<!--more-->

Pour bien suivre, commencez par ces guides si ce n'est pas déjà fait :

- [Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %})
- [Comment dockeriser une API FastAPI]({% post_url 2025-08-16-Comment-dockeriser-une-api-web-avec-FastAPI %})

Objectifs :

- Comprendre pourquoi et comment découper un projet FastAPI en plusieurs
  fichiers
- Utiliser `APIRouter` et `include_router` pour organiser les endpoints
- Mettre la route de statut `/info/status` dans un fichier dédié
- Préparer le terrain pour la configuration et l'extension de l'application

## Pourquoi séparer en plusieurs fichiers ?

- Lisibilité : chaque fichier a une responsabilité claire (routes, config,
  démarrage)
- Évolutivité : ajouter un nouveau domaine fonctionnel = ajouter un nouveau
  router
- Testabilité : des modules plus petits facilitent l'écriture de tests unitaires
- Ré-utilisabilité : mieux isolé, le code est plus facile à partager et à
  refactorer

## Structure de projet proposée

Voici une structure simple et efficace pour des APIs de taille petite à
moyenne :

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

- `app/main.py` contient l'instance FastAPI et inclut les routers
- `app/routers/status.py` définit le router pour `/info/status`
- `app/routers/exemple.py` définit le router pour `/exemple/*`
- `app/routers/__init__.py` peut rester vide, il marque le dossier comme package
  Python

> Remarque : si vous avez suivi l'article sur Docker, la route `/info/status`
> reste la même. Votre `Dockerfile` et sa healthcheck HTTP continueront de
> fonctionner sans modification.

## Implémentation pas à pas

### 1) Créer le router de statut (fichier dédié)

Créez `app/routers/status.py` :

```python
from fastapi import APIRouter

router = APIRouter(prefix="/info", tags=["status"])  # préfixe commun à toutes les routes de ce router

@router.get("/status")
def info_status():
    # Réponse adaptée aux checks de disponibilité (k8s, Docker, etc.)
    return {"status": "ok"}
```

### 2) Créer l'application et inclure le router

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

### 3) Marquer les dossiers comme packages Python

Créez des fichiers vides `app/__init__.py` et `app/routers/__init__.py` pour
permettre les imports relatifs (si ce n'est pas déjà le cas dans votre
environnement).

Contenu (peut rester vide) :

```python
# app/__init__.py
```

```python
# app/routers/__init__.py
```

## Lancer l'application

Depuis la racine du projet :

```bash
uvicorn app.main:app --reload
```

Par défaut, l'API sera disponible sur http://127.0.0.1:8000

- Test manuel :

```bash
curl http://127.0.0.1:8000/
# "Hello World"

curl http://127.0.0.1:8000/info/status
# {"status":"ok"}
```

- Documentation automatique :
  - Swagger UI : http://127.0.0.1:8000/docs
  - Redoc : http://127.0.0.1:8000/redoc

## Étendre l'application : ajouter d'autres routers

La force de cette approche est de pouvoir ajouter facilement de nouveaux
domaines fonctionnels.
Par exemple, vous pouvez créer `app/routers/users.py` avec son propre
`APIRouter(prefix="/users")` et l'inclure via `app.include_router(users_router)`
dans `create_app()`.

Cette organisation facilite :

- la séparation par domaine (users, items, auth, etc.)
- l'ajout de middlewares ou de dépendances au niveau d'un router
- la lecture et la maintenance du code sur le long terme

## Résumé

- Nous avons découpé l'application FastAPI en modules : un point d'entrée (
  `app/main.py`) et un fichier dédié pour la route de statut (
  `app/routers/status.py`).
- Nous avons utilisé `APIRouter` et `include_router` pour organiser et brancher
  les endpoints.
- Cette structure améliore la lisibilité et prépare l'application à grandir
  proprement.

## Voir aussi

- [Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %})
- [Comment dockeriser une API FastAPI]({% post_url 2025-08-16-Comment-dockeriser-une-api-web-avec-FastAPI %})
- [Limiter le rate d’une API FastAPI avec Redis (fastapi-limiter)]({% post_url 2025-09-20-Limiter-le-rate-d-une-API-FastAPI-avec-Redis %})
- [Comment faire une api web avec Flask]({% post_url 2021-04-20-Comment-faire-une-api-web-en-python %})
- [Comment faire des requêtes HTTP avec requests]({% post_url 2020-05-22-Comment-faire-des-requetes-http-en-python-avec-requests %})
- [La doc de FastAPI](https://fastapi.tiangolo.com/)
