---
layout: article
title: "Comment dockeriser une application FastAPI"
author: Pierre Chopinet
tags:
- python
- fastapi
- http
- api
- rest
- docker
- multistage
- alpine
- devops
---

Dans ce tutoriel, nous allons dockeriser une API web FastAPI avec un Dockerfile
multi-étapes : une première étape prépare les dépendances Python, la seconde
produit une image Alpine qui ne contient que ce qu'il faut pour faire tourner
l'API avec uvicorn.

<!--more-->

Dans cet article :
- Structure du projet
- Le Dockerfile multi-étapes
- La healthcheck
- Le fichier .dockerignore
- Construire et lancer l'image

Pré-requis : savoir créer une API FastAPI minimale (sinon, commencez par
[Python : Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %}))
et avoir installé Docker ([guide officiel](https://docs.docker.com/get-docker/)).

## Structure du projet

On part d'un projet minimal, avec trois fichiers à la racine :

```
.
├── app.py
├── requirements.txt
└── Dockerfile
```

Le fichier `app.py` contient l'application FastAPI, avec deux routes : `GET /`
pour le "hello world", et `GET /info/status` que Docker appellera pour vérifier
que l'API répond.

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def root():
    return "Hello World"

@app.get("/info/status")
def info_status():
    # Route appelée par la healthcheck du Dockerfile
    return {"status": "ok"}
```

Le fichier `requirements.txt` liste les dépendances :

```
fastapi
uvicorn[standard]
```

L'extra `standard` d'uvicorn installe notamment uvloop, une boucle
d'événements plus rapide que celle d'asyncio, et httptools pour analyser les
requêtes HTTP : uvicorn les utilise automatiquement quand ils sont présents.
Pour que deux builds donnent la même image, il vaut mieux fixer les versions
dans ce fichier (`pip freeze` donne celles de votre environnement).

## Le Dockerfile multi-étapes

Un Dockerfile multi-étapes contient plusieurs `FROM`. Chaque étape part de sa
propre image, et l'image finale ne garde que la dernière étape, plus ce qu'on y
copie explicitement depuis les précédentes. On s'en sert ici pour garder les
outils de compilation hors de l'image finale.

Créez ce `Dockerfile` à la racine du projet :

```dockerfile
# ============================================================================
# Étape 1 : builder, construit les wheels (.whl) des dépendances Python
# ----------------------------------------------------------------------------
FROM python:3.13.5-alpine3.22 AS builder

# Dossier de travail dans le conteneur (tous les chemins seront relatifs à /app)
WORKDIR /app

# Outils pour compiler les dépendances qui n'ont pas de wheel pour Alpine
# (build-base contient gcc, make et les en-têtes de la libc musl)
RUN apk add --no-cache \
    build-base \
    libffi-dev \
    openssl-dev

# On copie uniquement les dépendances pour profiter du cache Docker
COPY requirements.txt .

# On construit les wheels pour toutes les dépendances spécifiées
RUN pip wheel --no-cache-dir --wheel-dir /app/wheels -r requirements.txt

# ============================================================================
# Étape 2 : image finale, sans les outils de compilation
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

# Healthcheck : vérifie périodiquement que l'API répond sur /info/status.
# L'image Alpine fournit déjà wget (via busybox), donc pas besoin d'installer curl.
# Variante avec curl (si vous l'avez installé plus haut) :
#   CMD curl -fs http://127.0.0.1:8000/info/status | grep -q '"status":"ok"' || exit 1
HEALTHCHECK --interval=60s --timeout=10s --start-period=5s --retries=3 \
  CMD wget -qO- http://127.0.0.1:8000/info/status || exit 1

# Commande de lancement : Uvicorn sert l'application FastAPI (objet `app`)
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

La première étape s'appelle `builder` (`AS builder`). On y installe
`build-base` (gcc, make et les en-têtes de musl, la libc d'Alpine) et les
en-têtes de libffi et d'OpenSSL, utilisés par certaines dépendances écrites en
C. Puis `pip wheel` prépare une wheel, un paquet Python prêt à installer, pour
chaque dépendance. Inutile d'ajouter le paquet Alpine `python3-dev` : il
contient les en-têtes du Python 3.12 de la distribution, alors que l'image
officielle `python` embarque déjà ceux de son propre Python 3.13 dans
`/usr/local/include`.

Avec le `requirements.txt` de cet article, pip trouve des wheels déjà
compilées pour Alpine (musllinux) pour toutes les dépendances, et ne compile
rien (vérifié en octobre 2026). Les outils de compilation ne serviront que le
jour où vous ajouterez une dépendance sans wheel pour Alpine.

La seconde étape repart de l'image `python` d'origine : les compilateurs
restent dans l'étape `builder`. On y copie les wheels, et
`pip install --no-index --find-links` les installe sans aller chercher quoi que
ce soit sur PyPI. Les dépendances sont installées avant de copier le code : tant
que `requirements.txt` ne change pas, Docker réutilise ces couches en cache, et
une modification de `app.py` ne relance pas l'installation.

Attention, le `rm -rf /app/wheels` fait disparaître les wheels du système de
fichiers de l'image, mais ne la fait pas maigrir : elles restent stockées dans
la couche créée par le `COPY --from=builder` (une dizaine de Mo ici), une couche
suivante ne pouvant que les masquer. Pour s'en passer, BuildKit, le builder par
défaut de Docker, permet de monter le dossier de l'étape `builder` le temps d'un
`RUN`. On remplace alors le `COPY --from=builder` et le `RUN pip install` par :

```dockerfile
COPY requirements.txt .
RUN --mount=type=bind,from=builder,source=/app/wheels,target=/tmp/wheels \
    pip install --no-cache-dir --no-index --find-links=/tmp/wheels -r requirements.txt \
    && find /usr/local -type d -name __pycache__ -exec rm -rf {} +
```

Enfin, `CMD` lance uvicorn sur `0.0.0.0` : par défaut il n'écoute que sur
`127.0.0.1`, et l'API ne serait alors pas joignable depuis l'extérieur du
conteneur. Si votre application est découpée en plusieurs fichiers, comme dans
l'article
[Organiser une application FastAPI en plusieurs fichiers]({% post_url 2025-08-17-Organiser-une-application-FastAPI-en-plusieurs-fichiers %}),
adaptez le module à lancer (`app.main:app` au lieu de `app:app`).

## La healthcheck

L'instruction `HEALTHCHECK` demande à Docker d'appeler `/info/status` toutes
les 60 secondes. `wget`, fourni par busybox dans l'image Alpine, renvoie un code
d'erreur si l'API ne répond pas ou répond avec une erreur HTTP : pas besoin
d'installer curl. Après 3 échecs consécutifs (`--retries=3`), le conteneur est
marqué `unhealthy`.

Si vous préférez curl, décommentez la ligne `RUN apk add --no-cache curl` et
utilisez la variante donnée en commentaire, qui vérifie en plus le contenu de la
réponse.

## Le fichier .dockerignore

`COPY . .` copie tout le dossier du projet dans l'image. Pour ne pas y embarquer
un environnement virtuel, l'historique git ou un fichier `.env` contenant des
secrets, créez un fichier `.dockerignore` à côté du Dockerfile :

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

Depuis la racine du projet (là où se trouve le Dockerfile) :

```bash
docker build -t fastapi-app:latest .
```

On lance le conteneur en mappant son port 8000 sur le port 8000 de la machine :

```bash
docker run --rm -p 8000:8000 --name fastapi-app fastapi-app:latest
```

L'API répond alors sur `http://127.0.0.1:8000/`, avec sa documentation sur
`/docs` (Swagger UI) et `/redoc` (ReDoc) :

```bash
curl http://127.0.0.1:8000/
"Hello World"

curl http://127.0.0.1:8000/info/status
{"status":"ok"}
```

Pour voir l'état de la healthcheck :

{% raw %}
```bash
docker inspect --format='{{json .State.Health}}' fastapi-app | jq
```
{% endraw %}

Le champ `Status` vaut `starting` jusqu'au premier contrôle réussi, puis
`healthy`. Depuis Docker 27, pendant la `start-period`, Docker lance un contrôle
toutes les 5 secondes : le conteneur passe donc à `healthy` environ 5 secondes
après son démarrage. Avec une version plus ancienne, le premier contrôle n'a
lieu qu'au bout de l'`interval`, soit 60 secondes ici.

## Voir aussi

- [Python : Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %})
- [Organiser une application FastAPI en plusieurs fichiers]({% post_url 2025-08-17-Organiser-une-application-FastAPI-en-plusieurs-fichiers %})
- [Comment dockeriser une application flask]({% post_url 2023-02-10-Comment-dockeriser-une-application-flask %})
- [Comment dockeriser une application Django]({% post_url 2025-10-25-Comment-dockeriser-une-application-Django %})
- [Comment manipuler du JSON en ligne de commande avec jq]({% post_url 2025-09-17-Comment-utiliser-jq %})
- [FastAPI in Containers - Docker](https://fastapi.tiangolo.com/deployment/docker/), la documentation de FastAPI
- [La référence du Dockerfile](https://docs.docker.com/reference/dockerfile/)
