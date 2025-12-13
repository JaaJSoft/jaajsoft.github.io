---
layout: article
title: "Comment dockeriser une application Django"
author: Pierre Chopinet
tags:
  - python
  - django
  - docker
  - devops
  - tutoriel
---

Dans ce tutoriel, nous allons dockeriser une application Django pas à pas,
avec Gunicorn.
<!--more-->

Prérequis :
- Connaissances de base de Python et Django.
- Avoir un projet Django fonctionnel.

Plan de ce guide :

1. Préparer le projet Django (réglages prod, statiques)
2. Choisir les dépendances (gunicorn, whitenoise, etc.)
3. Écrire un Dockerfile propre
4. Lancer en prod avec Gunicorn
5. Avec docker-compose (Postgres)
6. Développement vs production : quoi changer ?

---

## 1) Préparer votre projet Django

Avant de dockeriser, assurez-vous d’avoir un projet Django qui démarre
localement. Pour la suite, supposons que votre module de configuration s’appelle
`config` (créé via `django-admin startproject config .`). Adaptez si nécessaire.

Réglages recommandés dans `config/settings.py` (ou `settings/production.py` si
vous avez une config par environnements) :

```python
# config/settings.py (extrait)
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY = os.getenv("SECRET_KEY", "dev-secret-key")
DEBUG = bool(int(os.getenv("DEBUG", "0")))
ALLOWED_HOSTS = os.getenv("ALLOWED_HOSTS", "*").split(",")

# Static files (servis par WhiteNoise en prod pour simplifier)
STATIC_URL = "/static/"
STATIC_ROOT = BASE_DIR / "staticfiles"

INSTALLED_APPS = [
    # ...
    "django.contrib.staticfiles",
]

MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    # WhiteNoise doit être placé tôt dans la chaîne
    "whitenoise.middleware.WhiteNoiseMiddleware",
    # ...
]

# Compression/Cache des statiques
STATICFILES_STORAGE = (
    "whitenoise.storage.CompressedManifestStaticFilesStorage"
)

# Base de données (simple: sqlite par défaut, Postgres via variables d'env)
if os.getenv("DATABASE_URL"):
    # Option 1: via dj-database-url (pratique)
    import dj_database_url
    DATABASES = {
        "default": dj_database_url.parse(os.getenv("DATABASE_URL"), conn_max_age=600)
    }
else:
    DATABASES = {
        "default": {
            "ENGINE": "django.db.backends.sqlite3",
            "NAME": BASE_DIR / "db.sqlite3",
        }
    }
```

Points clés :

- `SECRET_KEY`, `DEBUG`, `ALLOWED_HOSTS` doivent venir de l’environnement.
- `STATIC_ROOT` + WhiteNoise permettent de servir facilement les fichiers
  statiques sans Nginx. Pour un trafic élevé ou du contenu média, un Nginx ou un
  stockage externe (S3, GCS) reste conseillé.
- `DATABASE_URL` vous permet d’activer Postgres/MySQL en un seul env var
  (ex : `postgres://user:pass@db:5432/app`).

Avant le build, vérifiez que vos statiques se collectent correctement :

```bash
python manage.py collectstatic --noinput
```

---

## 2) Dépendances utiles

Ajoutez (au minimum) dans votre `requirements.txt` :

```
Django>=4.2
# Serveur WSGI de prod
gunicorn
# Statiques en prod sans nginx
whitenoise
# Parsing de DATABASE_URL (optionnel mais pratique)
dj-database-url
```

Si vous voulez utiliser Postgres :

```
psycopg[binary]
```

> Note : `psycopg` est la version 3 du connecteur Postgres pour Python. C’est la
recommandation pour les nouveaux projets. Django fonctionne nativement avec
`psycopg` sans changer l’ENGINE (laissez `django.db.backends.postgresql`). Si
vous avez déjà un historique avec psycopg2, `psycopg2-binary` continue de
fonctionner, mais migrez idéalement vers psycopg 3.


---

## 3) Dockerfile minimal

On va partir d’une image officielle Python « slim » (souvent plus simple que
alpine pour compiler certaines libs) et préparer un conteneur prêt pour Django.

```dockerfile
# Dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# Création d'un user non-root (bonne pratique)
RUN useradd -m appuser

WORKDIR /app

# Dépendances système minimales (et nettoyage)
RUN apt-get update \
    && apt-get install -y --no-install-recommends build-essential \
    && rm -rf /var/lib/apt/lists/*

# Installer les dépendances Python en amont pour profiter du cache Docker
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

# Copier le code
COPY . .

# Collecte des statiques à l'image (facilite le run)
# Ces variables seront surchargées en runtime via docker-compose/env
ENV DJANGO_SETTINGS_MODULE=config.settings \
    SECRET_KEY=build-secret \
    DEBUG=0 \
    ALLOWED_HOSTS="*"

# Nombre de workers Gunicorn paramétrable (valeur par défaut)
ENV GUNICORN_WORKERS=3

RUN python manage.py collectstatic --noinput

# Droits et port
RUN chown -R appuser:appuser /app
USER appuser
EXPOSE 8000

# Commande de démarrage (Gunicorn)
# Remplacez `config.wsgi:application` par votre chemin WSGI si besoin
# Forme shell pour interpoler ${GUNICORN_WORKERS}
CMD ["sh", "-c", "gunicorn config.wsgi:application -b 0.0.0.0:8000 -w ${GUNICORN_WORKERS}"]
```

> À propos de `-w` (workers Gunicorn)
> - L’option `-w 3` indique à Gunicorn de démarrer 3 processus « workers ».
> - Chaque worker traite les requêtes de façon indépendante ; augmenter le nombre de workers augmente la capacité de traitement concurrente (au prix d’un peu plus de mémoire).
> - Règle empirique souvent citée : `workers = 2 * CPU + 1`. Ce n’est qu’un point de départ : mesurez et ajustez selon votre charge (I/O, CPU) et votre budget RAM.
> - Trop peu de workers saturent vite votre app ; trop de workers consomment inutilement de la mémoire et peuvent nuire aux performances.

Notes :

- Pour SQLite, rien à installer côté système. Pour Postgres, `psycopg[binary]` (psycopg 3)
  suffit souvent avec l’image `slim`. Sur alpine, il faut en général ajouter
  `musl-dev`, `gcc`, `postgresql-dev`, etc. Si vous souhaitez la variante optimisée C, utilisez `psycopg[c]` et installez les dépendances de compilation adéquates.
- On exécute `collectstatic` au build pour accélérer le run. Vous pouvez aussi le
  faire à l’entrée du conteneur si vos pipelines l’exigent.
- On expose 8000 (Gunicorn), libre à vous de mapper vers 80/8080 côté hôte.

Pensez au `.dockerignore` (évite d’envoyer des fichiers inutiles) :

```
.git
.gitignore
__pycache__
*.pyc
.env
media/
node_modules/
```

---

## 4) Lancer l’image en production

Build :

```bash
docker build -t mon_app_django:latest .
```

Run (SQLite, sans DB externe) :

```bash
docker run -p 8000:8000 \
  -e SECRET_KEY="change-me" \
  -e DEBUG=0 \
  -e ALLOWED_HOSTS="localhost,127.0.0.1" \
  mon_app_django:latest
```

Vérifiez :

```bash
curl http://127.0.0.1:8000/
```

> Pour tester des endpoints applicatifs facilement, voir
[requests]({% post_url 2020-05-22-Comment-faire-des-requetes-http-en-python-avec-requests %}).

---

## 5) docker-compose avec Postgres

Pour une stack plus réaliste, utilisons Postgres via `docker-compose.yml` :

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:18-alpine
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      timeout: 3s
      retries: 5

  web:
    build: .
    environment:
      SECRET_KEY: "change-me"
      DEBUG: "0"
      ALLOWED_HOSTS: "localhost,127.0.0.1"
      DJANGO_SETTINGS_MODULE: "config.settings"
      DATABASE_URL: "postgres://app:app@db:5432/app"
      GUNICORN_WORKERS: "4"  # override du nombre de workers au run
    ports:
      - "8000:8000"
    depends_on:
      db:
        condition: service_healthy
    # On utilise la CMD du Dockerfile (forme shell) qui lit GUNICORN_WORKERS

volumes:
  pgdata:
```

Initialisez la base et un compte admin :

```bash
# créer/mettre à jour le schéma
docker compose run --rm web python manage.py migrate
# créer un superutilisateur
docker compose run --rm web python manage.py createsuperuser
# lancer les services
docker compose up -d
```

Vous pouvez maintenant accéder à votre app sur http://127.0.0.1:8000/ et à
l’admin Django sur http://127.0.0.1:8000/admin/.

---

## 6) Développement vs production

- Serveur de développement : pour un feedback instantané, vous pouvez lancer
  `runserver` et monter le code en volume :

  ```yaml
  # docker-compose.override.yml (exemple dev)
  services:
    web:
      command: ["python", "manage.py", "runserver", "0.0.0.0:8000"]
      environment:
        DEBUG: "1"
      volumes:
        - ./:/app
  ```

- Collecte des statiques : en dev, ce n’est pas indispensable. En prod, exécutez
  `collectstatic` (au build ou au déploiement) pour servir via WhiteNoise.

- Nginx : non indispensable dans ce guide. Pour la plupart des projets, vous
  pouvez démarrer sans, puis ajouter Nginx quand vous aurez besoin de TLS,
  d’assets lourds ou d’un reverse proxy multi‑apps.

- Rate limiting, cache, sécurité : au-delà de Django, pensez à votre
  infrastructure. Exemple côté FastAPI/Redis :
  [Rate limiter avec Redis]({% post_url 2025-09-20-Limiter-le-rate-d-une-API-FastAPI-avec-Redis %}).
  Et côté serveur, durcissez votre machine :
  [Installer et configurer Fail2ban]({% post_url 2025-09-21-Installer-et-configurer-Fail2ban-sur-Ubuntu-Debian %}).

---

## Dépannage (FAQ)

- Problème d’`ALLOWED_HOSTS` : en prod, mettez le FQDN (ex : `monapp.fr`). En
  dev, `localhost,127.0.0.1` suffisent.
- Fichiers statiques manquants (404) : assurez-vous d’avoir lancé
  `collectstatic`, que WhiteNoise est bien dans le `MIDDLEWARE` et que
  `STATIC_ROOT` existe.
- ImportError `config.wsgi` : adaptez le chemin de votre projet
  (`<nom_du_projet>.wsgi:application`) dans la commande Gunicorn.
- Postgres ne démarre pas : vérifiez `depends_on` et la santé du service `db`.

---

## Conclusion

Vous avez maintenant une base saine pour conteneuriser votre application Django,
avec un Dockerfile propre, un serveur WSGI de production, et une orchestration
simplifiée via docker-compose.

À partir d’ici, vous pouvez ajouter un reverse
proxy (Nginx, Traefik), une CD/CI, des workers asynchrones (Celery + Redis),
ainsi que du monitoring et de la journalisation centralisée.

## Voir aussi

- [Comment ajouter du cache à une application Django]({% post_url 2025-11-01-Comment-ajouter-du-cache-a-une-application-Django %})
- [Accélérer Django avec la compression GZip]({% post_url 2025-11-22-Accelerer-Django-avec-la-compression-GZip %})
- [Comment dockeriser une application Flask]({% post_url 2023-02-10-Comment-dockeriser-une-application-flask %})
- [Limiter le rate d'une API avec Redis (FastAPI)]({% post_url 2025-09-20-Limiter-le-rate-d-une-API-FastAPI-avec-Redis %})
- [Installer et configurer Fail2ban sur Ubuntu/Debian]({% post_url 2025-09-21-Installer-et-configurer-Fail2ban-sur-Ubuntu-Debian %})
- [Faire des requêtes HTTP en Python avec requests]({% post_url 2020-05-22-Comment-faire-des-requetes-http-en-python-avec-requests %})
