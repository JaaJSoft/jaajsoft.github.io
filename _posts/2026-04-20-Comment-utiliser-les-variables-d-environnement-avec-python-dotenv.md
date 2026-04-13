---
layout: article
title: "Python : Comment utiliser les variables d'environnement avec python-dotenv"
tags:
    - python
    - dotenv
    - environnement
    - configuration
    - devops
author: Pierre Chopinet
---

Gérer la configuration d'une application Python (clés d'API, identifiants de
base de données, mode debug…) directement dans le code source est une mauvaise
pratique. Les variables d'environnement permettent de séparer la configuration
du code, et `python-dotenv` rend cette gestion simple et efficace grâce aux
fichiers `.env`.
<!--more-->

C'est un outil incontournable pour tout projet Python, que ce soit une API
Flask/FastAPI, un script de traitement de données ou une application Django.
Si vous avez déjà dockerisé une application, vous avez probablement manipulé
des variables d'environnement : `python-dotenv` vous permet de les gérer de
la même manière en développement local.

Dans cet article, vous allez apprendre à :

- Comprendre pourquoi externaliser la configuration dans des variables d'environnement
- Installer et utiliser `python-dotenv`
- Écrire un fichier `.env` avec la bonne syntaxe
- Charger les variables dans votre code Python
- Gérer plusieurs environnements (dev, test, production)
- Protéger vos secrets en excluant `.env` du dépôt Git

---

## Pourquoi utiliser des variables d'environnement ?

Imaginons une application qui se connecte à une base de données :

```python
# ❌ À ne pas faire : les identifiants en dur dans le code
import psycopg2

conn = psycopg2.connect(
    host="localhost",
    database="mon_app",
    user="admin",
    password="motdepasse_secret"
)
```

Ce code pose plusieurs problèmes :

- **Sécurité** : le mot de passe est visible dans le dépôt Git par quiconque y a accès
- **Rigidité** : pour changer d'environnement (dev → production), il faut modifier le code
- **Collaboration** : chaque développeur doit modifier le fichier pour ses propres identifiants

La solution est de stocker ces valeurs dans des **variables d'environnement**.
C'est l'un des principes de l'application [twelve-factor](https://12factor.net/fr/config) :
la configuration doit être séparée du code.

## Installation

Installez `python-dotenv` avec pip :

```bash
pip install python-dotenv
```

> Si vous utilisez un environnement virtuel (ce qui est recommandé), activez-le
> avant d'installer le package.

## Créer un fichier .env

Le fichier `.env` se place à la racine de votre projet. Il contient les
variables sous la forme `CLÉ=valeur`, une par ligne :

```env
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

- Les lignes commençant par `#` sont des commentaires
- Les espaces autour du `=` sont autorisés mais déconseillés
- Les valeurs contenant des espaces doivent être entre guillemets : `APP_NAME="Mon Application"`
- Les variables peuvent référencer d'autres variables : `DATABASE_URL=postgres://${DB_USER}:${DB_PASSWORD}@${DB_HOST}:${DB_PORT}/${DB_NAME}`
- Les lignes vides sont ignorées

## Charger les variables dans Python

### Utilisation de base

La fonction `load_dotenv()` charge les variables du fichier `.env` dans les
variables d'environnement du processus. On y accède ensuite avec `os.getenv()` :

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

`os.getenv()` accepte un second paramètre qui sert de **valeur par défaut**
si la variable n'est pas définie. C'est utile pour les paramètres qui ont une
valeur raisonnable par défaut.

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

Par défaut, `load_dotenv()` **ne remplace pas** les variables d'environnement
déjà définies. Cela signifie que si `DB_HOST` est déjà défini dans
l'environnement système, la valeur du `.env` sera ignorée.

```python
# Les variables système ont la priorité
load_dotenv()  # DB_HOST du .env ignoré si déjà défini dans l'environnement
```

Ce comportement est voulu : en production, on définit les variables
d'environnement directement (via Docker, systemd, le cloud provider…), et le
fichier `.env` ne sert qu'en développement local.

Pour forcer le remplacement des variables existantes, utilisez le paramètre
`override` :

```python
# Forcer l'utilisation des valeurs du .env
load_dotenv(override=True)
```

> ⚠️ Utilisez `override=True` avec précaution. En production, cela pourrait
> écraser des variables d'environnement définies volontairement par
> l'infrastructure.

## Spécifier un fichier différent

Par défaut, `load_dotenv()` cherche un fichier `.env` dans le répertoire
courant, puis remonte l'arborescence. On peut aussi spécifier un chemin
explicitement :

```python
from dotenv import load_dotenv

# Charger un fichier spécifique
load_dotenv(".env.production")
```

C'est utile pour gérer **plusieurs environnements** avec des fichiers
distincts :

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

## Exemple concret : une connexion à une base de données

Voici un exemple complet qui montre comment structurer proprement la
configuration d'une connexion à PostgreSQL :

```env
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

Ce pattern de classe `Config` est très courant dans les projets Flask et
FastAPI. Il centralise toute la configuration au même endroit.

## Protéger ses secrets avec .gitignore

Le fichier `.env` contient des secrets. Il ne doit **jamais** être commité
dans le dépôt Git. Ajoutez-le à votre `.gitignore` :

```gitignore
# Variables d'environnement
.env
.env.*
!.env.example
```

En revanche, on peut créer un fichier `.env.example` (sans les vraies
valeurs) pour documenter les variables nécessaires :

```env
# .env.example — Copiez ce fichier en .env et remplissez les valeurs
DB_HOST=localhost
DB_PORT=5432
DB_NAME=
DB_USER=
DB_PASSWORD=
DEBUG=true
SECRET_KEY=
API_KEY=
```

Ce fichier `.env.example` peut être commité : il sert de documentation pour
les autres développeurs qui rejoignent le projet.

## Utilisation avec Docker

Si vous dockerisez votre application, `python-dotenv` s'intègre naturellement
dans le workflow. En développement, le fichier `.env` est chargé par
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

Dans les deux cas, le code Python reste identique grâce au comportement par
défaut de `load_dotenv()` qui ne remplace pas les variables déjà définies :

```python
load_dotenv()  # En dev : charge le .env / En Docker : les variables existent déjà
db_host = os.getenv("DB_HOST")
```

## Bonnes pratiques

> ✅ **À faire**
>
> - Toujours ajouter `.env` au `.gitignore`
> - Fournir un `.env.example` avec des valeurs vides ou d'exemple
> - Utiliser des valeurs par défaut raisonnables avec `os.getenv("CLÉ", "défaut")`
> - Centraliser la configuration dans un module dédié (`config.py`)
> - Valider les variables critiques au démarrage de l'application

> ❌ **À éviter**
>
> - Commiter le fichier `.env` dans le dépôt Git
> - Utiliser `override=True` en production
> - Mettre des valeurs de production dans le `.env.example`
> - Appeler `load_dotenv()` plusieurs fois sans raison

## Voir aussi

- [Comment dockeriser une application Flask]({% post_url 2023-02-10-Comment-dockeriser-une-application-flask %})
- [Comment dockeriser une application FastAPI]({% post_url 2025-08-16-Comment-dockeriser-une-api-web-avec-FastAPI %})
- [Comment dockeriser une application Django]({% post_url 2025-10-25-Comment-dockeriser-une-application-Django %})
- [Python : Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %})
- [Documentation officielle de python-dotenv](https://pypi.org/project/python-dotenv/)
