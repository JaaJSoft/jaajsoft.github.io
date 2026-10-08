---
layout: article
title: "Python : Comment utiliser les variables d'environnement avec python-dotenv"
description: "Gérer la configuration d'une application Python avec les variables d'environnement et python-dotenv : fichier .env, priorités, secrets et Docker."
tags:
    - python
    - dotenv
    - environnement
    - configuration
    - devops
author: Pierre Chopinet
---

Écrire la configuration d'une application Python (clés d'API, identifiants de
base de données, mode debug...) directement dans le code source pose vite
problème. Les variables d'environnement séparent la configuration du code, et
`python-dotenv` permet de les définir dans un fichier `.env` pendant le
développement.
<!--more-->

On retrouve `python-dotenv` dans tous les types de projets Python : API Flask
ou FastAPI, script de traitement de données, application Django. Si vous avez
déjà dockerisé une application, vous avez probablement manipulé des variables
d'environnement : avec `python-dotenv`, vous travaillez de la même manière en
développement local.

Dans cet article :
- Pourquoi utiliser des variables d'environnement ?
- Installation
- Créer un fichier .env
- Charger les variables dans Python
- Priorité des variables
- Spécifier un fichier différent
- Exemple concret : une connexion à une base de données
- Protéger ses secrets avec .gitignore
- Utilisation avec Docker

## Pourquoi utiliser des variables d'environnement ?

Imaginons une application qui se connecte à une base de données :

```python
# À ne pas faire : les identifiants en dur dans le code
import psycopg2

conn = psycopg2.connect(
    host="localhost",
    database="mon_app",
    user="admin",
    password="motdepasse_secret"
)
```

Le mot de passe est visible par toute personne qui a accès au dépôt Git. Pour
passer du développement à la production, il faut modifier le code, et chaque
développeur de l'équipe doit changer le fichier pour y mettre ses propres
identifiants.

On stocke plutôt ces valeurs dans des variables d'environnement. C'est l'un des
principes de la méthodologie [twelve-factor](https://12factor.net/fr/config) :
la configuration doit être séparée du code.

## Installation

Installez `python-dotenv` avec pip, de préférence dans un environnement
virtuel (pensez à l'activer avant) :

```bash
pip install python-dotenv
```

Les exemples de cet article ont été testés avec python-dotenv 1.2.4.

## Créer un fichier .env

Le fichier `.env` se place à la racine de votre projet. Il contient les
variables sous la forme `CLÉ=valeur`, une par ligne :

```
# Configuration de la base de données
DB_HOST=localhost
DB_PORT=5432
DB_NAME=mon_app
DB_USER=admin
DB_PASSWORD=motdepasse_secret

# Configuration de l'application
DEBUG=true
SECRET_KEY=ma-cle-secrete-tres-longue
API_KEY=sk-1234567890abcdef
```

Quelques règles de syntaxe à connaître :

- les lignes vides et celles qui commencent par `#` (les commentaires) sont ignorées ;
- python-dotenv accepte des espaces autour du `=`, mais un shell les refuse : mieux vaut s'en passer ;
- les guillemets ne sont pas obligatoires, même si la valeur contient des espaces, mais ils gardent le fichier lisible par un shell (`APP_NAME="Mon Application"`).

Une variable peut aussi faire référence à d'autres variables, avec la forme
`${NOM}` :

```
DATABASE_URL=postgres://${DB_USER}:${DB_PASSWORD}@${DB_HOST}:${DB_PORT}/${DB_NAME}
```

Seule la forme avec accolades est interprétée : `$DB_USER` reste tel quel.

## Charger les variables dans Python

### Utilisation de base

La fonction `load_dotenv()` lit le fichier `.env` et ajoute ses variables à
l'environnement du processus. On y accède ensuite avec `os.getenv()` :

```python
import os
from dotenv import load_dotenv

# Charger le fichier .env
load_dotenv()

# Lire les variables
db_host = os.getenv("DB_HOST")
db_port = int(os.getenv("DB_PORT", "5432"))
debug = os.getenv("DEBUG", "false").lower() == "true"

print(f"Connexion à {db_host}:{db_port}")
print(f"Mode debug : {debug}")
```

Avec le fichier `.env` précédent, on obtient :

```
Connexion à localhost:5432
Mode debug : True
```

`os.getenv()` accepte un second paramètre qui sert de valeur par défaut si la
variable n'est pas définie, comme `"5432"` pour le port. Notez aussi que les
variables d'environnement sont toujours des chaînes de caractères : c'est à
vous de les convertir, d'où le `int()` et la comparaison avec `"true"`.

Si le fichier `.env` n'existe pas, `load_dotenv()` ne lève pas d'erreur et
retourne simplement `False`. C'est ce comportement silencieux qui permet au même
code de fonctionner en local (où le `.env` est présent) et en production (où les
variables sont déjà injectées dans l'environnement par l'orchestrateur).

### Utilisation avec dotenv_values()

Si vous préférez ne pas modifier les variables d'environnement du processus,
`dotenv_values()` retourne un dictionnaire :

```python
from dotenv import dotenv_values

config = dotenv_values(".env")

db_host = config["DB_HOST"]
db_port = int(config.get("DB_PORT", "5432"))
```

Cette approche est pratique quand on veut isoler la configuration sans affecter
l'environnement global, par exemple dans des tests.

## Priorité des variables

Par défaut, `load_dotenv()` ne remplace pas les variables d'environnement
déjà définies. Si `DB_HOST` existe déjà dans l'environnement système, la
valeur du `.env` est ignorée.

```python
# Les variables système ont la priorité
load_dotenv()  # DB_HOST du .env ignoré si déjà défini dans l'environnement
```

Ce comportement est voulu : en production, on définit les variables
d'environnement directement (via Docker, systemd, le fournisseur cloud...), et le
fichier `.env` ne sert qu'en développement local.

Pour forcer le remplacement des variables existantes, utilisez le paramètre
`override` :

```python
# Forcer l'utilisation des valeurs du .env
load_dotenv(override=True)
```

Attention avec `override=True` en production : il peut écraser des variables
d'environnement définies volontairement par l'infrastructure.

## Spécifier un fichier différent

Par défaut, `load_dotenv()` cherche un fichier `.env` dans le dossier du script
Python qui l'appelle, puis dans les dossiers parents. On peut aussi lui donner
un chemin explicitement :

```python
from dotenv import load_dotenv

# Charger un fichier spécifique
load_dotenv(".env.production")
```

Ce mécanisme sert à gérer plusieurs environnements avec des fichiers distincts :

```text
mon_projet/
    .env                # Configuration par défaut (dev)
    .env.test           # Configuration pour les tests
    .env.production     # Configuration de production
    app.py
```

```python
import os
from dotenv import load_dotenv

env = os.getenv("APP_ENV", "development")

if env == "test":
    load_dotenv(".env.test")
elif env == "production":
    load_dotenv(".env.production")
else:
    load_dotenv()  # .env par défaut
```

Attention, contrairement à la recherche par défaut, un chemin relatif comme
`.env.test` part du répertoire courant, pas du dossier du script : il faut
lancer le programme depuis la racine du projet, ou construire un chemin
absolu avec `Path(__file__).parent / ".env.test"`.

La recherche par défaut est faite par la fonction `find_dotenv()`, que
`load_dotenv()` appelle quand on ne lui passe pas de chemin. On peut l'appeler
soi-même pour changer son comportement, par exemple pour chercher le `.env` à
partir du répertoire courant plutôt qu'à partir du script :

```python
from dotenv import load_dotenv, find_dotenv

load_dotenv(find_dotenv(usecwd=True))
```

## Exemple concret : une connexion à une base de données

Voici un exemple complet qui montre comment structurer la configuration d'une
connexion à PostgreSQL :

```
# .env
DB_HOST=localhost
DB_PORT=5432
DB_NAME=mon_app
DB_USER=admin
DB_PASSWORD=motdepasse_secret
```

```python
# config.py
import os
from dotenv import load_dotenv

load_dotenv()

class Config:
    DB_HOST = os.getenv("DB_HOST", "localhost")
    DB_PORT = int(os.getenv("DB_PORT", "5432"))
    DB_NAME = os.getenv("DB_NAME")
    DB_USER = os.getenv("DB_USER")
    DB_PASSWORD = os.getenv("DB_PASSWORD")

    @property
    def database_url(self):
        return (
            f"postgresql://{self.DB_USER}:{self.DB_PASSWORD}"
            f"@{self.DB_HOST}:{self.DB_PORT}/{self.DB_NAME}"
        )

config = Config()
```

```python
# app.py
from config import config

print(f"Connexion à : {config.database_url}")
```

Ce qui affiche :

```
Connexion à : postgresql://admin:motdepasse_secret@localhost:5432/mon_app
```

Une classe `Config` de ce genre est très courante dans les projets Flask et
FastAPI : toute la configuration est regroupée au même endroit. Pour une
variable obligatoire, comme le mot de passe, on peut utiliser
`os.environ["DB_PASSWORD"]` à la place de `os.getenv()` : il lève une
`KeyError` si la variable n'existe pas, et l'application s'arrête dès son
démarrage au lieu d'échouer plus tard.

Pour aller plus loin, avec du typage strict et une validation automatique au
démarrage, on peut utiliser [`pydantic-settings`](https://docs.pydantic.dev/latest/concepts/pydantic_settings/),
souvent associé à FastAPI. Pydantic se charge alors
de convertir chaque valeur dans le type déclaré (`int`, `bool`...) et de lever
une erreur explicite si une variable obligatoire est absente.

## Protéger ses secrets avec .gitignore

Le fichier `.env` contient des secrets. Il ne doit jamais être commité dans le
dépôt Git. Ajoutez-le à votre `.gitignore` :

```
# Variables d'environnement
.env
.env.*
!.env.example
```

En revanche, on peut créer un fichier `.env.example` (sans les vraies
valeurs) pour documenter les variables nécessaires :

```
# .env.example - Copiez ce fichier en .env et remplissez les valeurs
DB_HOST=localhost
DB_PORT=5432
DB_NAME=
DB_USER=
DB_PASSWORD=
DEBUG=true
SECRET_KEY=
API_KEY=
```

Ce fichier `.env.example` peut être commité (c'est le rôle de la ligne
`!.env.example` du `.gitignore`) : il sert de documentation pour les autres
développeurs qui rejoignent le projet. N'y mettez jamais de valeurs de
production.

## Utilisation avec Docker

Si vous dockerisez votre application, le même code fonctionne en local et dans
le conteneur. En développement, le fichier `.env` est chargé par
`python-dotenv`. En production, les variables sont injectées par Docker :

```yaml
# docker-compose.yml
services:
  app:
    build: .
    env_file:
      - .env
```

Ou directement via des variables d'environnement :

```yaml
services:
  app:
    build: .
    environment:
      - DB_HOST=db
      - DB_PORT=5432
      - DB_NAME=mon_app
```

Dans les deux cas, le code Python reste identique : dans le conteneur, les
variables viennent de Docker, et `load_dotenv()` ne trouve rien à charger (il
retourne `False` silencieusement). Cela suppose que le `.env` ne soit pas copié
dans l'image : avec un `COPY . .` dans le Dockerfile, ajoutez-le à votre
`.dockerignore`. En local comme dans le conteneur, `os.getenv("DB_HOST")`
récupère la bonne valeur :

```python
load_dotenv()  # En dev : charge le .env / En Docker : no-op, les variables sont déjà là
db_host = os.getenv("DB_HOST")
```

## Voir aussi

- [Comment dockeriser une application flask]({% post_url 2023-02-10-Comment-dockeriser-une-application-flask %})
- [Comment dockeriser une application FastAPI]({% post_url 2025-08-16-Comment-dockeriser-une-api-web-avec-FastAPI %})
- [Comment dockeriser une application Django]({% post_url 2025-10-25-Comment-dockeriser-une-application-Django %})
- [Python : Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %})
- [Documentation officielle de python-dotenv](https://pypi.org/project/python-dotenv/)
