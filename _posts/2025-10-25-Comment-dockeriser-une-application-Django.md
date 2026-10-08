---
layout: article
title: "Comment dockeriser une application Django"
description: "Dockeriser une application Django pas à pas avec Gunicorn : Dockerfile minimal, lancement en production et Docker Compose avec PostgreSQL."
tags:
  - python
  - django
  - docker
  - devops
author: Pierre Chopinet
---

Dans ce tutoriel, nous allons dockeriser une application Django pas à pas : réglages de production, image Docker avec Gunicorn, puis Docker Compose avec une base Postgres.
<!--more-->

Dans cet article :
- Préparer le projet Django
- Les dépendances
- Le Dockerfile
- Lancer l'image
- Docker Compose avec Postgres
- En développement

Pré-requis : connaître les bases de Django et avoir un projet qui démarre en local. Pour les bases de Docker, l'article [Comment dockeriser une application flask]({% post_url 2023-02-10-Comment-dockeriser-une-application-flask %}) suit la même démarche avec Flask.

Versions utilisées : Django 5.2 (LTS), Gunicorn 26, WhiteNoise 6.12, Python 3.12 et Postgres 18.

## Préparer le projet Django

Pour la suite, on suppose que le module de configuration s'appelle `config` (projet créé avec `django-admin startproject config .`). Adaptez les chemins si le vôtre porte un autre nom.

Dans un conteneur, la configuration passe par des variables d'environnement. Voici les réglages à adapter dans `config/settings.py` :

```python
# config/settings.py (extrait)
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY = os.getenv("SECRET_KEY", "dev-secret-key")
DEBUG = bool(int(os.getenv("DEBUG", "0")))
ALLOWED_HOSTS = os.getenv("ALLOWED_HOSTS", "*").split(",")

# Fichiers statiques, servis par WhiteNoise
STATIC_URL = "/static/"
STATIC_ROOT = BASE_DIR / "staticfiles"

INSTALLED_APPS = [
    # ...
    "django.contrib.staticfiles",
]

MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    # WhiteNoise juste après SecurityMiddleware
    "whitenoise.middleware.WhiteNoiseMiddleware",
    # ...
]

# Compression et noms versionnés pour les statiques
# (STORAGES remplace STATICFILES_STORAGE, supprimé en Django 5.1)
STORAGES = {
    "default": {
        "BACKEND": "django.core.files.storage.FileSystemStorage",
    },
    "staticfiles": {
        "BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage",
    },
}

# SQLite par défaut, Postgres (ou autre) si DATABASE_URL est définie
if os.getenv("DATABASE_URL"):
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

`SECRET_KEY`, `DEBUG` et `ALLOWED_HOSTS` viennent de l'environnement, avec des valeurs par défaut qui permettent de lancer le projet sans rien configurer (`DEBUG` est alors désactivé). Attention à la valeur `"*"` pour `ALLOWED_HOSTS` : elle accepte n'importe quel en-tête `Host`, ce qui revient à désactiver la vérification que fait Django contre les attaques par en-tête `Host`. En production, listez vos domaines : `ALLOWED_HOSTS=monapp.fr,www.monapp.fr`.

WhiteNoise permet à Gunicorn de servir lui-même les fichiers statiques, sans Nginx devant. Sa documentation demande de le placer juste après `SecurityMiddleware`, avant tous les autres middlewares. Le stockage `CompressedManifestStaticFilesStorage` ajoute une empreinte dans le nom des fichiers, pour que les navigateurs puissent les garder en cache indéfiniment. Il en crée aussi des versions compressées au moment du `collectstatic`. Par contre, WhiteNoise ne sert pas les fichiers envoyés par les utilisateurs (`media`) : il leur faut un stockage dédié, S3 ou GCS par exemple.

Avec `DATABASE_URL`, une seule variable suffit pour passer à Postgres ou MySQL, par exemple `postgres://user:pass@db:5432/app`.

Avant de construire l'image, vérifiez que la collecte des fichiers statiques fonctionne :

```bash
python manage.py collectstatic --noinput
```

Avec le stockage "Manifest", cette étape n'a rien d'optionnel : si elle n'a pas été faite, Django ne trouve pas le fichier `staticfiles.json` et, avec `DEBUG` à 0, chaque page qui fait référence à un fichier statique part en erreur 500 avec `ValueError: Missing staticfiles manifest entry for 'admin/css/base.css'`.

Enfin, `python manage.py check --deploy` passe en revue les réglages de sécurité attendus en production. Avec les valeurs par défaut ci-dessus, il signale par exemple que la clé `dev-secret-key` est trop faible (`security.W009`) et que les cookies de session et CSRF ne sont pas marqués `Secure`. Lancez-le avec les variables d'environnement de la production.

## Les dépendances

Le fichier `requirements.txt` doit contenir au minimum :

```
Django~=5.2.0
# Serveur WSGI de production
gunicorn
# Fichiers statiques sans Nginx
whitenoise
# Lecture de DATABASE_URL
dj-database-url
```

Pour Postgres, on ajoute le connecteur :

```
psycopg[binary]
```

`psycopg` est la version 3 du connecteur Postgres pour Python, celle que Django recommande. Elle utilise le même `ENGINE` que psycopg2 (`django.db.backends.postgresql`) : rien à changer dans les réglages. L'extra `[binary]` installe une version précompilée qui embarque libpq (y compris pour Alpine) : il n'y a donc rien à compiler. Un projet existant peut garder `psycopg2-binary`, mais la documentation de Django prévient que sa prise en charge sera sans doute retirée un jour.

## Le Dockerfile

On part de l'image officielle `python:3.12-slim`, basée sur Debian, souvent plus simple qu'Alpine quand une dépendance doit être compilée. Créez ce `Dockerfile` à la racine du projet :

```dockerfile
# Dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# Utilisateur sans privilèges pour faire tourner l'application
RUN useradd -m appuser

WORKDIR /app

# Outils de compilation (inutiles si toutes vos dépendances ont une wheel)
RUN apt-get update \
    && apt-get install -y --no-install-recommends build-essential \
    && rm -rf /var/lib/apt/lists/*

# Les dépendances d'abord, pour profiter du cache de Docker
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

# Puis le code
COPY . .

ENV DJANGO_SETTINGS_MODULE=config.settings \
    GUNICORN_WORKERS=3

# Collecte des fichiers statiques dans l'image
RUN python manage.py collectstatic --noinput

RUN chown -R appuser:appuser /app
USER appuser
EXPOSE 8000

# Remplacez config.wsgi:application par le chemin WSGI de votre projet
CMD ["sh", "-c", "exec gunicorn config.wsgi:application -b 0.0.0.0:8000 -w ${GUNICORN_WORKERS}"]
```

L'ordre des instructions compte : comme `requirements.txt` est copié et les dépendances installées avant le reste du code, Docker réutilise ces couches tant que `requirements.txt` ne change pas. Une modification du code ne relance donc pas le `pip install`.

Le paquet `build-essential` ne sert que si une de vos dépendances doit être compilée. Celles de cet article ont toutes une version précompilée (wheel), vous pouvez donc retirer ce `RUN` pour alléger l'image.

Le `collectstatic` est fait pendant le build, l'image contient donc déjà les fichiers statiques compressés. Avec les réglages précédents, la commande se contente des valeurs par défaut. Si vos réglages exigent une clé secrète (par exemple `SECRET_KEY = os.environ["SECRET_KEY"]`), passez une valeur factice à cette seule commande :

```dockerfile
RUN SECRET_KEY=factice python manage.py collectstatic --noinput
```

Évitez de la mettre dans un `ENV` : ces variables restent dans l'image (`docker inspect` les affiche) et s'appliqueraient au conteneur si vous oubliiez de passer la vraie valeur au lancement.

L'option `-b 0.0.0.0:8000` est nécessaire : par défaut, Gunicorn n'écoute que sur `127.0.0.1`, une adresse injoignable depuis l'extérieur du conteneur. Le mot-clé `exec` a aussi son importance. On passe par `sh -c` pour que `${GUNICORN_WORKERS}` soit remplacé par sa valeur, mais sans `exec`, le shell reste le processus principal du conteneur (le PID 1) et Gunicorn tourne en dessous. Or `docker stop` envoie son signal d'arrêt au PID 1 : le shell ne le transmet pas, Gunicorn ne s'arrête pas proprement et Docker finit par tuer le conteneur au bout de 10 secondes. Avec `exec`, Gunicorn remplace le shell et reçoit directement le signal.

Si votre module de configuration ne s'appelle pas `config`, pensez à adapter `config.wsgi:application`, sinon les workers ne démarrent pas et le log affiche `ModuleNotFoundError: No module named 'config'`.

L'option `-w` fixe le nombre de *workers*. Ce sont des processus indépendants, et avec le type de worker par défaut (synchrone), chacun traite une requête à la fois. Augmenter leur nombre permet de traiter plus de requêtes en parallèle, au prix de plus de mémoire. La documentation de Gunicorn conseille de partir de `2 * nombre de cœurs + 1` et d'ajuster selon la charge, en précisant que 4 à 12 workers suffisent en général, même pour un trafic important.

Nginx n'est pas obligatoire pour démarrer : WhiteNoise sert les statiques et Gunicorn le reste. Gunicorn recommande tout de même de placer devant ses workers synchrones un proxy comme Nginx, qui met en tampon les échanges avec les clients lents : sans lui, quelques connexions lentes suffisent à occuper tous les workers. Un reverse proxy devient aussi utile pour le TLS ou pour héberger plusieurs applications sur la même machine.

Pensez aussi au fichier `.dockerignore`, à côté du Dockerfile, pour ne pas envoyer dans l'image l'historique git, les secrets locaux ou votre base SQLite de développement :

```
.git
.gitignore
__pycache__
*.pyc
.env
.venv/
db.sqlite3
media/
node_modules/
```

## Lancer l'image

On construit l'image :

```bash
docker build -t mon_app_django:latest .
```

Puis on la lance en passant la configuration par l'environnement :

```bash
docker run -p 8000:8000 \
  -e SECRET_KEY="change-me" \
  -e DEBUG=0 \
  -e ALLOWED_HOSTS="localhost,127.0.0.1" \
  mon_app_django:latest
```

Sur un projet tout neuf, il n'y a pas de page d'accueil (la racine renvoie une 404), mais l'administration répond par une redirection vers sa page de connexion :

```bash
curl -I http://127.0.0.1:8000/admin/
```

Vous devriez obtenir quelque chose comme :

```
HTTP/1.1 302 Found
Server: gunicorn
Date: Wed, 07 Oct 2026 20:14:22 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Location: /admin/login/?next=/admin/
Expires: Wed, 07 Oct 2026 20:14:22 GMT
Cache-Control: max-age=0, no-cache, no-store, must-revalidate, private
X-Frame-Options: DENY
Content-Length: 0
Vary: Cookie
X-Content-Type-Options: nosniff
Referrer-Policy: same-origin
Cross-Origin-Opener-Policy: same-origin
```

Ici, les migrations ne sont pas appliquées et la base SQLite disparaît avec le conteneur : c'est suffisant pour vérifier que l'image démarre, pas pour s'en servir. Pour une vraie base, on passe à Docker Compose.

## Docker Compose avec Postgres

On décrit les deux services, l'application et Postgres, dans un fichier `docker-compose.yml` :

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
      # Depuis Postgres 18, le volume officiel est /var/lib/postgresql
      # (avant la 18, on montait /var/lib/postgresql/data)
      - pgdata:/var/lib/postgresql
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
      GUNICORN_WORKERS: "4"  # remplace la valeur par défaut du Dockerfile
    ports:
      - "8000:8000"
    depends_on:
      db:
        condition: service_healthy
    # Pas de command : on garde la CMD du Dockerfile, qui lit GUNICORN_WORKERS

volumes:
  pgdata:
```

Le `healthcheck` utilise `pg_isready` pour savoir quand Postgres accepte les connexions, et `condition: service_healthy` fait attendre le service `web` jusque-là.

Attention au volume : depuis Postgres 18, l'image officielle déclare son volume sur `/var/lib/postgresql` et range les données dans un sous-dossier par version (`/var/lib/postgresql/18/docker`). Avec une version 17 ou plus ancienne, il faut monter `/var/lib/postgresql/data`, sinon les données ne survivent pas à la recréation du conteneur.

On crée ensuite le schéma et un compte administrateur, puis on démarre le tout :

```bash
# Créer ou mettre à jour le schéma
docker compose run --rm web python manage.py migrate
# Créer un superutilisateur
docker compose run --rm web python manage.py createsuperuser
# Lancer les services
docker compose up -d
```

`docker compose run` démarre au passage le service `db` dont dépend `web`, et `--rm` supprime le conteneur une fois la commande terminée. L'application répond maintenant sur http://127.0.0.1:8000/ et l'administration sur http://127.0.0.1:8000/admin/.

## En développement

Pour développer avec le rechargement automatique du code, on remplace Gunicorn par `runserver` et on monte le dossier du projet dans le conteneur. Ces changements vont dans un fichier `docker-compose.override.yml` :

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

Docker Compose lit automatiquement ce fichier en plus de `docker-compose.yml` lors d'un `docker compose up` : il doit donc rester sur les postes de développement et ne jamais se retrouver sur le serveur de production. Avec `DEBUG` à 1, `runserver` sert lui-même les fichiers statiques : le `collectstatic` n'est pas nécessaire.

## Voir aussi

- [Comment dockeriser une application flask]({% post_url 2023-02-10-Comment-dockeriser-une-application-flask %})
- [Comment dockeriser une application FastAPI]({% post_url 2025-08-16-Comment-dockeriser-une-api-web-avec-FastAPI %})
- [Accélérer Django avec la compression HTTP]({% post_url 2025-12-13-Accelerer-Django-avec-la-compression-GZip %})
- [Comment ajouter du cache à une application Django]({% post_url 2025-11-01-Comment-ajouter-du-cache-a-une-application-Django %})
- [Checklist de déploiement de Django](https://docs.djangoproject.com/en/5.2/howto/deployment/checklist/)
- [Documentation de Gunicorn](https://gunicorn.org/)
- [Documentation de WhiteNoise](https://whitenoise.readthedocs.io/)
