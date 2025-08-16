---
layout: article
title: "Comment dockeriser une API FastAPI"
author: Pierre Chopinet
tags:
- python
- fastapi
- api
- docker
- multistage
- healthcheck
- tutoriel
- devops
---

Dans ce tutoriel, nous allons apprendre à dockeriser une API web développée avec FastAPI, en utilisant un Dockerfile multi‑étapes (builder + image finale Alpine) optimisé pour la taille et la vitesse d'installation. <!--more-->

Objectifs :

- Construire une image légère et reproductible
- Comprendre le multi‑stage build (wheels Python en étape de build)
- Ajouter une healthcheck HTTP vers l'endpoint `/info/status`
- Démarrer l'API avec `uvicorn`

Pré‑requis : savoir créer une API FastAPI minimale. Si ce n'est pas encore fait, suivez d'abord ce guide:

[Python : Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %})

## Structure minimale du projet

À la racine de votre projet, vous pouvez partir d'une structure très simple :

```
.
├── app.py
├── requirements.txt
└── Dockerfile
```

- `app.py` contient l'application FastAPI exposée via une variable `app`.
- `requirements.txt` liste les dépendances Python.
- `Dockerfile` décrit la construction de l'image Docker.

### app.py

Nous allons créer deux endpoints : `GET /` pour un "hello world" et `GET /info/status` qui renvoie un JSON attendu par la healthcheck du Dockerfile.

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def root():
    return "Hello World"

@app.get("/info/status")
def info_status():
    # La healthcheck du Dockerfile vérifie la présence exacte de "\"status\":\"ok\""
    return {"status": "ok"}
```

### requirements.txt

```
fastapi
uvicorn[standard]
```

> Remarque: `uvicorn[standard]` installe les extras recommandés (uvloop, httptools,…) pour de meilleures performances.

## Dockerfile (multi‑étapes)

Copiez‑collez ce Dockerfile à la racine du projet. Il construit d'abord des wheels (étape builder) puis installe ces wheels dans une image finale propre et compacte.

```dockerfile
# ============================================================================
# Étape 1 : Builder — construire les wheels (.whl) des dépendances Python
# Objectif : isoler la compilation pour accélérer les builds suivants et obtenir
# une image finale plus légère et plus propre.
# ----------------------------------------------------------------------------
FROM python:3.13.5-alpine3.22 AS builder

# Dossier de travail dans le conteneur (tous les chemins seront relatifs à /app)
WORKDIR /app

# Dépendances système nécessaires pour compiler certaines libs Python (c-extensions)
# Paquets: build-base, gcc, musl-dev, python3-dev, libffi-dev, openssl-dev
RUN apk add --no-cache \
    build-base \
    gcc \
    musl-dev \
    python3-dev \
    libffi-dev \
    openssl-dev

# On copie uniquement les dépendances pour profiter du cache Docker
COPY requirements.txt .

# On construit les wheels pour toutes les dépendances spécifiées
RUN pip wheel --no-cache-dir --wheel-dir /app/wheels -r requirements.txt

# ============================================================================
# Étape 2 : Image finale (runtime) — minimale et prête à exécuter
# ----------------------------------------------------------------------------
FROM python:3.13.5-alpine3.22

# Dossier de travail
WORKDIR /app

# (Optionnel) Installer curl si vous utilisez la healthcheck basée sur curl
# RUN apk add --no-cache curl

# On copie les wheels produits par l'étape builder
COPY --from=builder /app/wheels /app/wheels

# On installe les dépendances à partir des wheels locaux (pas d'accès réseau)
COPY requirements.txt .
RUN pip install --no-cache-dir --no-index --find-links=/app/wheels -r requirements.txt \
    && rm -rf /app/wheels \
    && rm -rf /root/.cache/pip \
    && find /usr/local -type d -name __pycache__ -exec rm -rf {} +

# On copie le code de l'application (app.py, etc.)
COPY . .

# On expose le port d'écoute de l'API dans le conteneur
EXPOSE 8000

# Healthcheck : vérifie périodiquement que l'API renvoie {"status":"ok"}
# Si curl n'est pas installé, remplacez par :
#   wget -qO- http://localhost:8000/info/status | grep -q '"status":"ok"'
HEALTHCHECK --interval=60s --timeout=10s --start-period=5s --retries=3 \
  CMD response=$(curl -s http://localhost:8000/info/status) && echo "$response" | grep -q '"status":"ok"' || exit 1

# Commande de lancement : Uvicorn sert l'application FastAPI (objet `app`)
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

> Note importante : la healthcheck utilise `curl`. Si votre image de base ne contient pas `curl`, vous pouvez soit l'ajouter dans l'étape finale (`RUN apk add --no-cache curl`), soit remplacer la commande par `wget -qO- http://localhost:8000/info/status | grep -q '"status":"ok"'`.

## (Optionnel) .dockerignore

Pour éviter d'embarquer des fichiers inutiles dans le build, créez un `.dockerignore` :

```
__pycache__
*.pyc
*.pyo
*.pyd
.env
.venv
venv
.git
.gitignore
.DS_Store
.idea
.vscode

# Dossiers de build
build
wheels
```

## Construire et lancer l'image

Depuis la racine du projet (là où se trouve le Dockerfile) :

```bash
docker build -t fastapi-app:latest .
```

Lancez le conteneur en mappant le port 8000 :

```bash
docker run --rm -p 8000:8000 --name fastapi-app fastapi-app:latest
```

Vous pouvez maintenant tester :

- Navigateur : http://127.0.0.1:8000/
- Swagger UI : http://127.0.0.1:8000/docs
- ReDoc : http://127.0.0.1:8000/redoc

Via `curl` :

```bash
curl http://127.0.0.1:8000/
# Hello World

curl -s http://127.0.0.1:8000/info/status
# {"status":"ok"}
```

Pour vérifier la healthcheck Docker :

```bash
docker inspect --format='{{json .State.Health}}' fastapi-app | jq
# Vous devriez voir Status: healthy après quelques secondes si /info/status renvoie {"status":"ok"}
```

## Aller plus loin

- Article précédent : [Python : Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %})
- Ajoutez un reverse proxy (Nginx, Traefik) devant votre API.
- Utilisez des variables d'environnement et des secrets.
- Intégrez un CI/CD pour builder et pousser automatiquement vos images.

Bon build et bonne mise en prod !
