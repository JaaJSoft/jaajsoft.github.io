---
layout: article
title: "Python : Mettre en cache des fonctions avec lru_cache"
tags:
  - python
  - performance
  - functools
  - cache
  - optimisation
author: Pierre Chopinet
---

Quand une fonction est coûteuse à exécuter (calcul lourd, requête réseau, lecture de fichier) et qu'on l'appelle plusieurs fois avec les mêmes arguments, on peut éviter de refaire le travail à chaque fois en mettant son résultat en cache. Python fournit pour cela le décorateur `@lru_cache` dans le module `functools`, qui transforme n'importe quelle fonction en version mémoïsée en une ligne.
<!--more-->

C'est l'outil le plus simple pour accélérer une fonction sans toucher à son code : on ajoute le décorateur, et les appels suivants avec les mêmes arguments retournent instantanément la valeur précédemment calculée. Cet article couvre `@lru_cache` mais aussi ses cousins de `functools` (`@cache`, `@cached_property`) et les pièges à éviter.

Dans cet article, vous allez apprendre à :

- Comprendre le principe de la mémoïsation
- Utiliser `@lru_cache` pour accélérer une fonction
- Inspecter, vider et dimensionner le cache
- Utiliser `@cache` et `@cached_property` à bon escient
- Identifier les limitations et les pièges courants

Pré-requis : être à l'aise avec les fonctions Python et les décorateurs.

---

## Pourquoi mettre en cache une fonction ?

Prenons une fonction qui calcule la suite de Fibonacci de manière récursive :

```python
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)
```

Le problème de cette implémentation, c'est qu'elle recalcule plusieurs fois les mêmes valeurs. Pour `fibonacci(30)`, la fonction est appelée plus de 2,7 millions de fois alors que seules 31 valeurs distinctes existent. Le temps d'exécution explose vite : `fibonacci(35)` prend déjà plusieurs secondes.

En ajoutant un cache, on stocke chaque résultat dès la première fois, et les appels suivants retournent instantanément la valeur :

```python
from functools import lru_cache

@lru_cache
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)
```

`fibonacci(35)` passe alors de plusieurs secondes à quelques microsecondes. La fonction reste identique, seul le décorateur change.

Cette technique s'appelle la **mémoïsation** : on garde en mémoire les résultats d'une fonction pour éviter de la rappeler avec les mêmes entrées.

## Un premier exemple complet

Voici un exemple plus parlant : une fonction qui simule un calcul lent.

```python
import time
from functools import lru_cache

@lru_cache(maxsize=100)
def calcul_lent(x):
    print(f"Calcul de {x}...")
    time.sleep(2)
    return x * x

# Premier appel : 2 secondes
print(calcul_lent(4))
# Calcul de 4...
# 16

# Deuxième appel avec la même entrée : instantané
print(calcul_lent(4))
# 16

# Nouvelle entrée : 2 secondes à nouveau
print(calcul_lent(5))
# Calcul de 5...
# 25
```

Le premier appel exécute la fonction normalement. Les appels suivants avec les mêmes arguments retournent le résultat depuis le cache sans réexécuter le corps de la fonction.

## Comment fonctionne LRU ?

LRU signifie **Least Recently Used**, "le moins récemment utilisé". Le cache a une taille maximale (par défaut 128 entrées) et quand cette taille est atteinte, l'entrée la moins récemment utilisée est supprimée pour faire de la place à la nouvelle.

C'est un compromis raisonnable entre mémoire consommée et taux de réussite du cache. Si vos appels sont concentrés sur un petit nombre d'arguments fréquents, LRU les garde en mémoire et évince les arguments rares.

## Les paramètres de lru_cache

### maxsize

Contrôle le nombre maximal d'entrées dans le cache.

```python
@lru_cache(maxsize=256)
def ma_fonction(x):
    ...
```

- `maxsize=128` (défaut) : taille raisonnable pour la plupart des cas
- `maxsize=None` : cache illimité, sans éviction. À utiliser uniquement si l'espace des entrées possibles est borné.
- `maxsize=0` : désactive le cache (utile pour comparer les performances)

### typed

Si `typed=True`, les arguments de types différents sont mis en cache séparément, même s'ils sont égaux :

```python
@lru_cache(typed=True)
def f(x):
    return x

f(3)    # mis en cache
f(3.0)  # mis en cache séparément, car float != int
```

Par défaut, `typed=False` et `f(3)` / `f(3.0)` partagent la même entrée puisque `3 == 3.0`.

## Inspecter et vider le cache

`@lru_cache` ajoute deux méthodes utiles à la fonction décorée.

### cache_info()

Retourne un `namedtuple` avec les statistiques du cache :

```python
@lru_cache(maxsize=100)
def carre(x):
    return x * x

carre(2)
carre(3)
carre(2)
print(carre.cache_info())
# CacheInfo(hits=1, misses=2, maxsize=100, currsize=2)
```

- `hits` : nombre d'appels où le résultat venait du cache
- `misses` : nombre d'appels où la fonction a vraiment été exécutée
- `maxsize` : taille maximale du cache
- `currsize` : nombre d'entrées actuellement stockées

Le ratio `hits / (hits + misses)` est le taux de réussite du cache. Plus il est élevé, plus le cache est efficace.

### cache_clear()

Vide entièrement le cache :

```python
carre.cache_clear()
print(carre.cache_info())
# CacheInfo(hits=0, misses=0, maxsize=100, currsize=0)
```

Utile dans les tests, après avoir modifié des données sous-jacentes, ou pour libérer de la mémoire.

## @cache : la version simplifiée

Depuis Python 3.9, `functools.cache` est un raccourci pour `@lru_cache(maxsize=None)`, c'est-à-dire un cache sans limite de taille :

```python
from functools import cache

@cache
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)
```

Il est légèrement plus rapide que `@lru_cache(maxsize=None)` car il n'a pas de logique d'éviction LRU à maintenir. À utiliser quand on sait que le nombre d'entrées distinctes restera raisonnable (par exemple : récursivité avec un domaine borné).

> ⚠️ Avec `@cache`, rien n'évite que le cache grossisse indéfiniment. Si vos arguments sont très variés, préférez `@lru_cache` avec un `maxsize` explicite.

## @cached_property : pour les propriétés calculées

`@cached_property` (Python 3.8+) est l'équivalent pour les méthodes d'instance qui calculent un attribut dérivé. Le résultat est calculé une seule fois par instance, à la première lecture :

```python
from functools import cached_property

class Document:
    def __init__(self, contenu):
        self.contenu = contenu

    @cached_property
    def nombre_mots(self):
        print("Calcul du nombre de mots...")
        return len(self.contenu.split())

doc = Document("ceci est un document de test")
print(doc.nombre_mots)
# Calcul du nombre de mots...
# 6

print(doc.nombre_mots)
# 6 (pas de recalcul)
```

Contrairement à `@lru_cache`, le résultat est stocké directement sur l'instance (dans `doc.__dict__`). Quand l'instance est libérée par le ramasse-miettes, sa valeur cachée disparaît avec elle : pas de fuite mémoire.

Pour invalider la valeur, il suffit de la supprimer :

```python
del doc.nombre_mots  # force le recalcul au prochain accès
```

## Les arguments doivent être hashables

Le cache est implémenté avec un dictionnaire, et les clés de dictionnaire doivent être hashables. Cela exclut les types mutables comme `list`, `dict` ou `set` :

```python
@lru_cache
def somme(items):
    return sum(items)

somme((1, 2, 3))  # OK : tuple est hashable
somme([1, 2, 3])  # TypeError: unhashable type: 'list'
```

Pour contourner cette limite, on convertit l'argument en type hashable (par exemple un `tuple` ou un `frozenset`) avant l'appel :

```python
@lru_cache
def somme_unique(items):
    return sum(items)

# On passe par un tuple
ma_liste = [1, 2, 3, 2, 1]
total = somme_unique(tuple(ma_liste))
```

## Attention aux méthodes d'instance

Utiliser `@lru_cache` sur une méthode d'instance fonctionne, mais avec un piège important : le cache contient une référence à `self`, ce qui empêche le ramasse-miettes de libérer l'instance tant que le cache existe.

```python
class Calculateur:
    @lru_cache
    def calcul(self, x):
        return x * 2
```

Sur le long terme, cela peut créer une fuite mémoire si vous créez de nombreuses instances. Deux alternatives :

1. Utiliser `@cached_property` si la valeur ne dépend que de l'instance.
2. Déplacer la méthode en fonction libre (en passant les attributs nécessaires en arguments), puis appliquer `@lru_cache` dessus.

## @lru_cache et les fonctions async

Le décorateur `@lru_cache` ne fonctionne pas correctement avec les fonctions asynchrones : il met en cache la **coroutine** retournée par l'appel, pas son résultat. Une coroutine ne pouvant être attendue qu'une seule fois, le deuxième `await` lèvera une exception.

```python
@lru_cache
async def fetch(url):  # ❌ piège
    ...
```

Pour mettre en cache une fonction `async`, utilisez une bibliothèque dédiée comme `async-lru` ou `aiocache`.

## Le cache est local au processus

`@lru_cache` stocke ses entrées dans la mémoire du processus Python. Cela implique plusieurs choses :

- Le cache est perdu à chaque redémarrage de l'application
- Plusieurs processus (par exemple plusieurs workers Gunicorn) ne partagent pas leur cache
- Les accès concurrents sont protégés par un verrou interne (`@lru_cache` est thread-safe)

Si vous avez besoin d'un cache partagé entre processus ou persistant, regardez du côté de Redis, de `cachetools`, ou d'une couche de cache applicative (voir les articles sur le cache Flask, Django et FastAPI plus bas).

## Cas d'usage pratiques

### Mémoïsation d'algorithmes récursifs

L'usage classique : transformer une récursivité exponentielle en récursivité polynomiale.

```python
from functools import cache

@cache
def combinaisons(n, k):
    if k == 0 or k == n:
        return 1
    return combinaisons(n - 1, k - 1) + combinaisons(n - 1, k)
```

Sans cache, `combinaisons(30, 15)` déclencherait des millions d'appels redondants. Avec `@cache`, chaque couple `(n, k)` est calculé une seule fois.

### Appels coûteux : I/O ou requêtes externes

Une fonction qui interroge une API peut bénéficier énormément du cache, à condition que les données ne changent pas pendant la session :

```python
import requests
from functools import lru_cache

@lru_cache(maxsize=512)
def fetch_user(user_id):
    response = requests.get(f"https://api.exemple.com/users/{user_id}")
    return response.json()
```

Si plusieurs parties du code demandent le même utilisateur, on évite les allers-retours réseau.

> ⚠️ Si les données peuvent changer en cours d'exécution, le cache renverra une version périmée. Dans ce cas, prévoyez un mécanisme d'invalidation ou utilisez un cache avec TTL (par exemple `cachetools.TTLCache`).

### Factory ou parsing

Quand on doit construire un objet coûteux à partir d'une clé, et que la clé revient souvent :

```python
import re
from functools import lru_cache

@lru_cache
def compile_regex(pattern):
    return re.compile(pattern)
```

Compiler une regex est rapide mais pas gratuit. Mettre la fonction de compilation en cache évite de recompiler les mêmes patterns à chaque utilisation.

## Bonnes pratiques

### ✅ À faire

- Utiliser `@cache` ou `@lru_cache(maxsize=None)` quand le domaine d'entrées est borné (récursivité avec petites valeurs)
- Spécifier un `maxsize` explicite si les entrées peuvent être très variées
- Vérifier `cache_info()` régulièrement pour valider que le cache est efficace
- Préférer `@cached_property` pour les attributs calculés sur une instance
- Convertir les arguments mutables en types hashables (`tuple`, `frozenset`) si nécessaire

### ❌ À éviter

- Mettre `@lru_cache` sur une méthode d'instance dans un code qui crée beaucoup d'objets (risque de fuite mémoire)
- Utiliser `@cache` sur une fonction dont les arguments sont très variés (cache illimité, mémoire qui grossit)
- Mettre en cache des fonctions qui ont des effets de bord (écriture en base, envoi d'email, modification d'un fichier...)
- Compter sur le cache pour des données qui peuvent changer en cours d'exécution sans mécanisme d'invalidation
- Décorer une fonction `async` avec `@lru_cache` (ça mettra en cache la coroutine, pas le résultat)

## Voir aussi

- [Python : Comment utiliser les décorateurs]({% post_url 2026-05-14-Python-les-decorateurs %})
- [Comment utiliser un cache avec Flask]({% post_url 2025-09-14-Comment-utiliser-un-cache-avec-Flask %})
- [Comment ajouter du cache à une application Django]({% post_url 2025-11-01-Comment-ajouter-du-cache-a-une-application-Django %})
- [Utiliser fastapi-cache2 avec FastAPI]({% post_url 2025-08-18-Utiliser-fastapi-cache2-avec-FastAPI %})
- [Documentation officielle de `functools`](https://docs.python.org/fr/3/library/functools.html)
