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

Un décorateur permet d'ajouter un comportement à une fonction (afficher un log, mesurer sa durée, vérifier des droits...) sans toucher à son code. Si vous avez déjà écrit `@staticmethod`, `@property` ou `@app.route("/")` avec Flask, vous en avez déjà utilisé : nous allons voir comment ils fonctionnent, puis comment écrire les vôtres.
<!--more-->

Dans cet article :
- Les fonctions sont des objets
- Un premier décorateur
- Préserver le nom et la docstring avec `functools.wraps`
- Mesurer le temps d'exécution
- Journaliser les appels
- Un décorateur avec des paramètres
- Contrôler l'accès à une fonction
- Empiler des décorateurs
- Les décorateurs de la bibliothèque standard
- Décorer une classe

Pré-requis : être à l'aise avec les fonctions en Python. Les exemples ont été testés avec Python 3.13.

## Les fonctions sont des objets

Pour comprendre les décorateurs, il faut d'abord se rappeler qu'en Python, les fonctions sont des objets comme les autres. On peut les affecter à une variable ou les passer en paramètre à une autre fonction :

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

Une fonction peut aussi en créer une autre et la retourner :

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

La fonction `saluer` créée à l'intérieur garde accès à `formule`, même une fois l'appel à `creer_salutation` terminé (on parle de *closure*). Les décorateurs reposent sur ce mécanisme.

## Un premier décorateur

Un décorateur est une fonction qui prend une fonction en paramètre et retourne une nouvelle fonction, qui appelle en général la première. Voici un décorateur qui affiche un message avant et après l'appel :

```python
def mon_decorateur(func):
    def wrapper(*args, **kwargs):
        print(f"Avant l'appel de {func.__name__}")
        resultat = func(*args, **kwargs)
        print(f"Après l'appel de {func.__name__}")
        return resultat
    return wrapper
```

Le `wrapper` accepte `*args, **kwargs` et les transmet tels quels à `func` : le décorateur fonctionne ainsi avec n'importe quelle signature. Il ne faut pas oublier non plus de retourner le résultat de `func`, sinon la fonction décorée renverrait toujours `None`.

On peut appliquer ce décorateur à la main :

```python
def dire_bonjour(nom):
    print(f"Bonjour {nom} !")

dire_bonjour = mon_decorateur(dire_bonjour)
dire_bonjour("Alice")
# Avant l'appel de dire_bonjour
# Bonjour Alice !
# Après l'appel de dire_bonjour
```

La syntaxe `@` fait la même chose, en plus lisible :

```python
@mon_decorateur
def dire_bonjour(nom):
    print(f"Bonjour {nom} !")

dire_bonjour("Alice")
# Avant l'appel de dire_bonjour
# Bonjour Alice !
# Après l'appel de dire_bonjour
```

Écrire `@mon_decorateur` au-dessus de la fonction revient à écrire `dire_bonjour = mon_decorateur(dire_bonjour)` juste après sa définition.

## Préserver le nom et la docstring avec `functools.wraps`

Notre décorateur a un défaut : la fonction décorée est maintenant le `wrapper`, elle a donc perdu son nom et sa docstring.

```python
@mon_decorateur
def dire_bonjour(nom):
    """Salue une personne par son nom."""
    print(f"Bonjour {nom} !")

print(dire_bonjour.__name__)  # wrapper (au lieu de dire_bonjour)
print(dire_bonjour.__doc__)   # None (au lieu de la docstring)
```

Ce n'est pas qu'un détail : `help()`, les outils de documentation et certains frameworks s'appuient sur ces informations. Avec Flask par exemple, le nom de la fonction sert de nom d'*endpoint* par défaut. Si deux routes utilisent un décorateur écrit sans `wraps`, les deux fonctions s'appellent `wrapper`, et Flask lève une erreur dès l'enregistrement de la seconde route : `View function mapping is overwriting an existing endpoint function: wrapper` (testé avec Flask 3.1).

Pour corriger ça, on utilise le décorateur `functools.wraps`, qui recopie sur le `wrapper` le nom, la docstring, le module et les annotations de la fonction d'origine :

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

`wraps` ajoute aussi un attribut `__wrapped__` qui pointe vers la fonction d'origine : `dire_bonjour.__wrapped__("Alice")` l'appelle sans passer par le décorateur. Prenez l'habitude de le mettre dans tous vos décorateurs, c'est ce que font les exemples suivants.

## Mesurer le temps d'exécution

Premier exemple concret, un décorateur qui affiche le temps d'exécution d'une fonction :

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
```

La durée dépend évidemment de la machine, on obtient par exemple :

```
traiter_donnees a pris 0.0974s
```

## Journaliser les appels

Sur le même modèle, ce décorateur écrit dans les logs chaque appel de la fonction, avec ses arguments et la valeur retournée :

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
# INFO:__main__:Appel de calculer_total((10, 5), {'remise': 0.1})
# INFO:__main__:calculer_total a retourné 45.0
```

On peut ainsi suivre les appels de plusieurs fonctions sans ajouter de `logger.info` dans chacune d'elles.

## Un décorateur avec des paramètres

Pour configurer un décorateur, il faut un niveau d'imbrication de plus : une fonction qui reçoit les paramètres et retourne le décorateur.

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

Quand Python rencontre `@repeter(3)`, il appelle d'abord `repeter(3)`, qui retourne le décorateur, puis il applique ce décorateur à la fonction.

Un exemple plus utile : réessayer une fonction quand elle lève une exception, typiquement un appel réseau qui échoue de temps en temps.

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

Après la dernière tentative, le `raise` sans argument relance l'exception d'origine : l'appelant voit l'erreur, au lieu de recevoir un `None` sans explication.

## Contrôler l'accès à une fonction

Les paramètres servent aussi à écrire des décorateurs de vérification. Celui-ci refuse d'exécuter la fonction si l'utilisateur n'a pas le rôle demandé :

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
supprimer_utilisateur(viewer, 42)   # PermissionError: Rôle 'admin' requis, rôle actuel : 'viewer'
```

Ici, le `wrapper` attend l'utilisateur en premier argument : ce décorateur ne s'applique qu'aux fonctions qui respectent cette convention.

## Empiler des décorateurs

On peut appliquer plusieurs décorateurs à une même fonction. Ils sont appliqués de bas en haut, en commençant par le plus proche de la fonction :

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

L'ordre compte. Ici, `timer` enveloppe la fonction produite par `retry` : il mesure donc la durée totale, nouvelles tentatives et temps d'attente compris. Dans l'ordre inverse, `timer` n'afficherait que la durée de la tentative qui a réussi, car une tentative qui échoue interrompt son `wrapper` avant le `print`.

## Les décorateurs de la bibliothèque standard

Avant d'écrire votre propre décorateur, vérifiez que la bibliothèque standard n'en fournit pas déjà un. Voici ceux qu'on croise le plus souvent.

### @property

`@property` transforme une méthode en attribut calculé, qu'on lit sans parenthèses. Avec un *setter*, on peut aussi contrôler les valeurs affectées :

```python
import math

class Cercle:
    def __init__(self, rayon):
        self.rayon = rayon  # passe par le setter

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
        return math.pi * self._rayon ** 2

c = Cercle(5)
print(c.rayon)  # 5 (pas de parenthèses)
print(c.aire)   # 78.53981633974483
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
print(d.jour, d.mois, d.annee)    # 15 6 2026
print(Date.est_bissextile(2024))  # True
```

Une méthode `@classmethod` reçoit la classe (`cls`) en premier paramètre au lieu de l'instance. On s'en sert surtout pour écrire des constructeurs alternatifs, comme `depuis_string`. Une méthode `@staticmethod` ne reçoit ni l'instance ni la classe : c'est une simple fonction rangée dans la classe.

### @functools.lru_cache

`@lru_cache` garde en mémoire le résultat de la fonction pour chaque combinaison d'arguments :

```python
from functools import lru_cache

@lru_cache(maxsize=128)
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

print(fibonacci(100))  # 354224848179261915075 (instantané grâce au cache)
```

Sans le cache, `fibonacci(100)` ne se terminerait pas en un temps raisonnable, car les mêmes valeurs seraient recalculées un nombre exponentiel de fois. Avec `@lru_cache`, chaque valeur n'est calculée qu'une fois. Depuis Python 3.9, `@functools.cache` fait la même chose sans limite de taille, comme `@lru_cache(maxsize=None)`. Le sujet est détaillé dans l'article sur [`lru_cache`]({% post_url 2026-05-25-Python-lru_cache %}).

### @dataclasses.dataclass

`@dataclass` génère `__init__`, `__repr__` et `__eq__` à partir des annotations de la classe :

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

## Décorer une classe

Un décorateur peut aussi s'appliquer à une classe : il reçoit la classe et retourne ce qu'il veut à la place. `@dataclass` retourne la même classe, complétée de nouvelles méthodes. Le décorateur suivant, lui, remplace la classe par une fonction qui renvoie toujours la même instance, un *singleton* :

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

Attention, après `@singleton`, `Configuration` n'est plus une classe mais une fonction : `isinstance(config1, Configuration)` lève une `TypeError`, et on ne peut plus en hériter. Si vous avez besoin de garder une vraie classe, il vaut mieux redéfinir `__new__` ou passer par une métaclasse.

## Voir aussi

- [Python : Mettre en cache des fonctions avec lru_cache]({% post_url 2026-05-25-Python-lru_cache %})
- [Python : Comment tester son code avec pytest]({% post_url 2026-04-06-Comment-tester-son-code-python-avec-pytest %})
- [Python : Comment faire une api web avec Flask]({% post_url 2021-04-20-Comment-faire-une-api-web-en-python %})
- [Python : Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %})
- [Python : Comment créer une CLI]({% post_url 2025-12-28-Comment-creer-une-CLI-en-python %})
- [Le terme "decorator" dans le glossaire Python](https://docs.python.org/3/glossary.html#term-decorator)
- [PEP 318 - Decorators for Functions and Methods](https://peps.python.org/pep-0318/)
- [Documentation du module functools](https://docs.python.org/3/library/functools.html)
