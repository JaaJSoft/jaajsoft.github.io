---
layout: article
title: "Python : Comment utiliser les décorateurs"
tags:
  - python
  - decorateurs
  - functools
  - metaprogrammation
author: Pierre Chopinet
---

Les décorateurs sont une fonctionnalité puissante de Python qui permet de modifier le comportement d'une fonction ou d'une classe sans toucher à son code. Si vous avez déjà utilisé `@staticmethod`, `@property` ou `@app.route("/")` dans Flask, vous avez déjà utilisé des décorateurs.
<!--more-->

Dans cet article :
- Ce qu'est un décorateur et comment il fonctionne
- Écrire ses propres décorateurs
- Décorateurs avec paramètres
- Décorateurs sur les classes
- Les décorateurs les plus utiles de la bibliothèque standard
- Cas d'usage pratiques

Pré-requis : être à l'aise avec les fonctions en Python, notamment le fait qu'une fonction est un objet comme un autre.

---

## Les fonctions sont des objets

Avant de comprendre les décorateurs, il faut comprendre un point fondamental de Python : les fonctions sont des objets. On peut les assigner à des variables, les passer en paramètre, et les retourner depuis d'autres fonctions.

```python
def saluer(nom):
    return f"Bonjour {nom} !"

# Assigner une fonction à une variable
ma_fonction = saluer
print(ma_fonction("Alice"))  # Bonjour Alice !

# Passer une fonction en paramètre
def executer(func, arg):
    return func(arg)

print(executer(saluer, "Bob"))  # Bonjour Bob !
```

Une fonction peut aussi retourner une autre fonction :

```python
def creer_salutation(formule):
    def saluer(nom):
        return f"{formule} {nom} !"
    return saluer

bonjour = creer_salutation("Bonjour")
coucou = creer_salutation("Coucou")

print(bonjour("Alice"))  # Bonjour Alice !
print(coucou("Bob"))     # Coucou Bob !
```

C'est exactement ce mécanisme qui est à la base des décorateurs.

---

## Un premier décorateur

Un décorateur est une fonction qui prend une fonction en paramètre et retourne une nouvelle fonction. Voici un décorateur qui affiche un message avant et après l'appel :

```python
def mon_decorateur(func):
    def wrapper(*args, **kwargs):
        print(f"Avant l'appel de {func.__name__}")
        resultat = func(*args, **kwargs)
        print(f"Après l'appel de {func.__name__}")
        return resultat
    return wrapper
```

On peut l'utiliser manuellement :

```python
def dire_bonjour(nom):
    print(f"Bonjour {nom} !")

dire_bonjour = mon_decorateur(dire_bonjour)
dire_bonjour("Alice")
# Avant l'appel de dire_bonjour
# Bonjour Alice !
# Après l'appel de dire_bonjour
```

La syntaxe `@` est un raccourci pour cette opération :

```python
@mon_decorateur
def dire_bonjour(nom):
    print(f"Bonjour {nom} !")

dire_bonjour("Alice")
# Avant l'appel de dire_bonjour
# Bonjour Alice !
# Après l'appel de dire_bonjour
```

`@mon_decorateur` au-dessus de la fonction est strictement équivalent à `dire_bonjour = mon_decorateur(dire_bonjour)`.

---

## Préserver les métadonnées avec functools.wraps

Il y a un problème avec le décorateur précédent : la fonction décorée perd son nom et sa docstring :

```python
@mon_decorateur
def dire_bonjour(nom):
    """Salue une personne par son nom."""
    print(f"Bonjour {nom} !")

print(dire_bonjour.__name__)  # wrapper (au lieu de dire_bonjour)
print(dire_bonjour.__doc__)   # None (au lieu de la docstring)
```

La solution est d'utiliser `functools.wraps` :

```python
from functools import wraps

def mon_decorateur(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print(f"Avant l'appel de {func.__name__}")
        resultat = func(*args, **kwargs)
        print(f"Après l'appel de {func.__name__}")
        return resultat
    return wrapper

@mon_decorateur
def dire_bonjour(nom):
    """Salue une personne par son nom."""
    print(f"Bonjour {nom} !")

print(dire_bonjour.__name__)  # dire_bonjour
print(dire_bonjour.__doc__)   # Salue une personne par son nom.
```

> Utilisez toujours `@wraps(func)` dans vos décorateurs. C'est une bonne habitude qui évite des surprises avec les outils de debugging, la documentation et les frameworks.

---

## Exemple pratique : mesurer le temps d'exécution

Un cas d'usage classique est de mesurer le temps d'exécution d'une fonction :

```python
import time
from functools import wraps

def timer(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        debut = time.perf_counter()
        resultat = func(*args, **kwargs)
        duree = time.perf_counter() - debut
        print(f"{func.__name__} a pris {duree:.4f}s")
        return resultat
    return wrapper

@timer
def traiter_donnees(n):
    total = sum(range(n))
    return total

traiter_donnees(10_000_000)
# traiter_donnees a pris 0.1842s
```

---

## Décorateur avec paramètres

Parfois, on veut configurer le comportement du décorateur. Pour cela, il faut ajouter un niveau d'imbrication supplémentaire : une fonction qui retourne le décorateur.

```python
from functools import wraps

def repeter(n):
    def decorateur(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(n):
                resultat = func(*args, **kwargs)
            return resultat
        return wrapper
    return decorateur

@repeter(3)
def dire_bonjour(nom):
    print(f"Bonjour {nom} !")

dire_bonjour("Alice")
# Bonjour Alice !
# Bonjour Alice !
# Bonjour Alice !
```

Quand Python voit `@repeter(3)`, il appelle d'abord `repeter(3)` qui retourne le décorateur, puis applique ce décorateur à la fonction.

### Autre exemple : retry

Un décorateur qui réessaie une fonction en cas d'exception :

```python
import time
from functools import wraps

def retry(max_tentatives=3, delai=1):
    def decorateur(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for tentative in range(1, max_tentatives + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if tentative == max_tentatives:
                        raise
                    print(f"Tentative {tentative} échouée : {e}. "
                          f"Nouvel essai dans {delai}s...")
                    time.sleep(delai)
        return wrapper
    return decorateur

@retry(max_tentatives=3, delai=2)
def appeler_api():
    # peut lever une exception
    ...
```

---

## Empiler des décorateurs

On peut appliquer plusieurs décorateurs sur une même fonction. Ils sont appliqués de bas en haut :

```python
@timer
@retry(max_tentatives=3, delai=1)
def appeler_api():
    ...
```

C'est équivalent à :

```python
appeler_api = timer(retry(max_tentatives=3, delai=1)(appeler_api))
```

L'ordre compte : ici, `timer` mesure le temps total incluant les retries.

---

## Décorateurs de la bibliothèque standard

Python fournit plusieurs décorateurs utiles dans sa bibliothèque standard.

### @property

Transforme une méthode en attribut accessible sans parenthèses :

```python
class Cercle:
    def __init__(self, rayon):
        self._rayon = rayon

    @property
    def rayon(self):
        return self._rayon

    @rayon.setter
    def rayon(self, valeur):
        if valeur < 0:
            raise ValueError("Le rayon doit être positif")
        self._rayon = valeur

    @property
    def aire(self):
        return 3.14159 * self._rayon ** 2

c = Cercle(5)
print(c.rayon)  # 5 (pas de parenthèses)
print(c.aire)   # 78.53975
c.rayon = 10    # passe par le setter
# c.rayon = -1  # ValueError
```

### @staticmethod et @classmethod

```python
class Date:
    def __init__(self, jour, mois, annee):
        self.jour = jour
        self.mois = mois
        self.annee = annee

    @classmethod
    def depuis_string(cls, date_string):
        jour, mois, annee = map(int, date_string.split("/"))
        return cls(jour, mois, annee)

    @staticmethod
    def est_bissextile(annee):
        return annee % 4 == 0 and (annee % 100 != 0 or annee % 400 == 0)

d = Date.depuis_string("15/06/2026")
print(Date.est_bissextile(2024))  # True
```

- `@classmethod` : reçoit la classe (`cls`) en premier paramètre, utile pour les constructeurs alternatifs
- `@staticmethod` : ne reçoit ni `self` ni `cls`, c'est une fonction utilitaire rattachée à la classe

### @functools.lru_cache

Met en cache les résultats d'une fonction selon ses arguments :

```python
from functools import lru_cache

@lru_cache(maxsize=128)
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

print(fibonacci(100))  # 354224848179261915075 (instantané grâce au cache)
```

Sans le cache, `fibonacci(100)` prendrait un temps astronomique à cause des appels récursifs redondants. Avec `@lru_cache`, chaque valeur est calculée une seule fois.

> Depuis Python 3.9, `@functools.cache` est un raccourci pour `@lru_cache(maxsize=None)` (cache illimité).

### @dataclasses.dataclass

Génère automatiquement `__init__`, `__repr__` et `__eq__` à partir des annotations :

```python
from dataclasses import dataclass

@dataclass
class Produit:
    nom: str
    prix: float
    stock: int = 0

p = Produit("Clavier", 49.99, 10)
print(p)  # Produit(nom='Clavier', prix=49.99, stock=10)
```

---

## Décorateur sur une classe

Un décorateur peut aussi s'appliquer à une classe entière. C'est exactement ce que fait `@dataclass` :

```python
from functools import wraps

def singleton(cls):
    instances = {}

    @wraps(cls)
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]

    return get_instance

@singleton
class Configuration:
    def __init__(self):
        self.debug = False
        self.version = "1.0"

config1 = Configuration()
config2 = Configuration()
print(config1 is config2)  # True (même instance)
```

> Attention : après `@singleton`, `Configuration` est devenue une fonction, pas
> une classe. `isinstance(config1, Configuration)` lève donc une `TypeError`.
> Si vous avez besoin de conserver le comportement de classe (pour `isinstance`,
> l'héritage, etc.), il vaut mieux surcharger `__new__` ou utiliser une
> métaclasse.

---

## Cas d'usage : contrôle d'accès

Un décorateur qui vérifie qu'un utilisateur a le bon rôle avant d'exécuter une fonction :

```python
from functools import wraps

def require_role(role):
    def decorateur(func):
        @wraps(func)
        def wrapper(utilisateur, *args, **kwargs):
            if utilisateur.get("role") != role:
                raise PermissionError(
                    f"Rôle '{role}' requis, "
                    f"rôle actuel : '{utilisateur.get('role')}'"
                )
            return func(utilisateur, *args, **kwargs)
        return wrapper
    return decorateur

@require_role("admin")
def supprimer_utilisateur(utilisateur, user_id):
    print(f"Utilisateur {user_id} supprimé")

admin = {"nom": "Alice", "role": "admin"}
viewer = {"nom": "Bob", "role": "viewer"}

supprimer_utilisateur(admin, 42)    # Utilisateur 42 supprimé
supprimer_utilisateur(viewer, 42)   # PermissionError
```

---

## Cas d'usage : logging automatique

```python
import logging
from functools import wraps

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def log_appel(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        logger.info(f"Appel de {func.__name__}({args}, {kwargs})")
        resultat = func(*args, **kwargs)
        logger.info(f"{func.__name__} a retourné {resultat}")
        return resultat
    return wrapper

@log_appel
def calculer_total(prix, quantite, remise=0):
    return prix * quantite * (1 - remise)

calculer_total(10, 5, remise=0.1)
# INFO:Appel de calculer_total((10, 5), {'remise': 0.1})
# INFO:calculer_total a retourné 45.0
```

---

## Bonnes pratiques

### À faire

- **Toujours utiliser `@wraps(func)`** pour préserver le nom et la docstring
- **Utiliser `*args, **kwargs`** dans le wrapper pour que le décorateur fonctionne avec n'importe quelle signature
- **Garder les décorateurs simples** : un décorateur = une responsabilité
- **Utiliser les décorateurs de la bibliothèque standard** (`@property`, `@lru_cache`, `@dataclass`) avant d'écrire les vôtres

### À ne pas faire

- **Modifier les arguments silencieusement** : un décorateur qui change les paramètres sans que l'appelant le sache crée des bugs difficiles à trouver
- **Mettre de la logique métier complexe** dans un décorateur : si ça dépasse 10-15 lignes, c'est probablement mieux dans une fonction normale
- **Empiler trop de décorateurs** : au-delà de 2 ou 3, le comportement devient difficile à suivre

---

## Conclusion

Les décorateurs permettent d'ajouter du comportement à des fonctions et des classes de manière réutilisable et non-intrusive. Le pattern est toujours le même : une fonction qui prend une fonction, l'enveloppe, et retourne le wrapper.

**Points clés :**
- Un décorateur est une fonction qui prend et retourne une fonction
- `@decorateur` est un raccourci pour `func = decorateur(func)`
- Toujours utiliser `@wraps(func)` pour préserver les métadonnées
- Pour les paramètres, ajouter un niveau d'imbrication supplémentaire
- La bibliothèque standard en fournit plusieurs : `@property`, `@lru_cache`, `@dataclass`, `@staticmethod`, `@classmethod`

---

## Pour aller plus loin

- [Documentation officielle sur les décorateurs](https://docs.python.org/3/glossary.html#term-decorator)
- [PEP 318 - Decorators for Functions and Methods](https://peps.python.org/pep-0318/)
- [Documentation functools](https://docs.python.org/3/library/functools.html)

## Voir aussi

- [Python : Comment tester son code avec pytest]({% post_url 2026-04-06-Comment-tester-son-code-python-avec-pytest %})
- [Python : Comment créer une CLI]({% post_url 2025-12-28-Comment-creer-une-CLI-en-python %})
- [Python : Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %})
- [Python : Comment faire une api web avec Flask]({% post_url 2021-04-20-Comment-faire-une-api-web-en-python %})
