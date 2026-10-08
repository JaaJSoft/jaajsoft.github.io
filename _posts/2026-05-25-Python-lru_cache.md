---
layout: article
title: "Python : Mettre en cache des fonctions avec lru_cache"
description: "Mettre en cache le résultat d'une fonction Python avec @lru_cache : principe LRU, taille du cache, @cache, @cached_property et cas où le cache pose problème."
tags:
  - python
  - performance
  - functools
  - cache
  - optimisation
author: Pierre Chopinet
---

Quand une fonction coûteuse (calcul lourd, requête réseau, lecture de fichier) est appelée plusieurs fois avec les mêmes arguments, on peut garder son résultat en mémoire au lieu de refaire le travail à chaque appel. En Python, le décorateur `@lru_cache` du module `functools` s'en charge en une ligne.
<!--more-->

Nous allons voir comment l'utiliser et le dimensionner, ses variantes `@cache` et `@cached_property`, et les cas où il vaut mieux s'en passer.

Dans cet article :
- Pourquoi mettre en cache une fonction ?
- Un exemple avec un calcul lent
- Le principe du cache LRU
- Inspecter et vider le cache
- `@cache`, un cache sans limite de taille
- `@cached_property` pour les attributs calculés
- Les arguments doivent être hashables
- Les méthodes d'instance
- Les fonctions async
- Le cache est local au processus
- Quelles fonctions mettre en cache

Pré-requis : être à l'aise avec les fonctions et les [décorateurs]({% post_url 2026-05-14-Python-les-decorateurs %}) en Python. Les exemples ont été testés avec Python 3.13 (`@cache` demande Python 3.9 ou plus récent).

## Pourquoi mettre en cache une fonction ?

Prenons la suite de Fibonacci, calculée de manière récursive :

```python
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)
```

Cette implémentation recalcule sans arrêt les mêmes valeurs : pour `fibonacci(30)`, la fonction est appelée près de 2,7 millions de fois alors qu'il n'existe que 31 valeurs distinctes. Le temps d'exécution explose vite : `fibonacci(35)` prend déjà environ une seconde avec Python 3.13, et chaque incrément de `n` multiplie ce temps par 1,6 environ.

Avec un cache, chaque résultat est stocké la première fois qu'il est calculé, et les appels suivants le récupèrent directement :

```python
from functools import lru_cache

@lru_cache
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)
```

Le premier appel à `fibonacci(35)` ne prend plus que quelques dizaines de microsecondes, et les suivants moins d'une microseconde puisque le résultat est déjà en cache. La fonction n'a pas changé, on a seulement ajouté le décorateur. Cette technique s'appelle la mémoïsation : on garde en mémoire les résultats d'une fonction pour ne pas la rappeler avec les mêmes entrées.

## Un exemple avec un calcul lent

Pour bien voir quand la fonction est réellement exécutée, voici une fonction qui simule un calcul de deux secondes :

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

Le deuxième appel avec l'argument `4` renvoie le résultat depuis le cache, sans exécuter le corps de la fonction : le message "Calcul de 4..." n'apparaît pas.

## Le principe du cache LRU

LRU signifie *Least Recently Used*, "le moins récemment utilisé". Le cache a une taille maximale, 128 entrées par défaut. Quand elle est atteinte, l'entrée qui n'a pas servi depuis le plus longtemps est supprimée pour faire de la place. Si les appels se concentrent sur un petit nombre d'arguments fréquents, ce sont eux qui restent en mémoire.

La taille se règle avec le paramètre `maxsize` :

```python
@lru_cache(maxsize=256)
def ma_fonction(x):
    ...
```

Avec `maxsize=None`, il n'y a plus de limite et le cache ne fait que grossir (c'est le comportement de `@cache`, présenté plus bas). Avec `maxsize=0`, le cache est désactivé, ce qui peut servir à comparer les performances avec et sans.

Le second paramètre, `typed`, concerne les arguments égaux mais de types différents. Par défaut (`typed=False`), ils sont en général considérés comme le même appel :

```python
@lru_cache
def ajouter(a, b):
    return a + b

print(ajouter(1, 2))      # 3
print(ajouter(1.0, 2.0))  # 3, et non 3.0 : c'est le résultat en cache
```

Avec `@lru_cache(typed=True)`, chaque combinaison de types a sa propre entrée, et le second appel renvoie bien `3.0`. La documentation précise que certains types comme `int` et `str` peuvent être mis en cache séparément même avec `typed=False` : pour une fonction à un seul argument, `f(3)` et `f(3.0)` occupent par exemple deux entrées différentes.

## Inspecter et vider le cache

`@lru_cache` ajoute notamment deux méthodes à la fonction décorée. `cache_info()` renvoie les statistiques du cache :

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

`hits` compte les appels servis par le cache, `misses` ceux où la fonction a vraiment été exécutée, `maxsize` est la taille maximale et `currsize` le nombre d'entrées stockées. Le rapport `hits / (hits + misses)` donne le taux de réussite du cache : s'il reste bas, le cache ne sert pas à grand-chose, ou `maxsize` est trop petit.

`cache_clear()` vide le cache et remet les compteurs à zéro :

```python
carre.cache_clear()
print(carre.cache_info())
# CacheInfo(hits=0, misses=0, maxsize=100, currsize=0)
```

C'est utile dans les tests, pour repartir d'un cache vide, ou quand les données dont dépend la fonction ont changé. Enfin, la fonction d'origine reste accessible par l'attribut `__wrapped__` (`carre.__wrapped__(4)`), pour l'appeler sans passer par le cache.

## `@cache`, un cache sans limite de taille

Depuis Python 3.9, `functools.cache` est un raccourci pour `@lru_cache(maxsize=None)` :

```python
from functools import cache

@cache
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)
```

C'est un simple alias, dont le code se résume à `return lru_cache(maxsize=None)(user_function)`. Les deux écritures donnent donc exactement le même cache, avec les mêmes performances et les mêmes méthodes `cache_info()` et `cache_clear()`. Comme il n'a jamais à supprimer d'entrée, ce cache sans limite est un peu plus léger et plus rapide qu'un `@lru_cache` avec un `maxsize`, qui doit tenir à jour l'ordre d'utilisation des entrées.

En contrepartie, rien ne l'empêche de grossir indéfiniment. Réservez `@cache` aux fonctions dont le nombre d'arguments distincts reste raisonnable, par exemple une récursion sur un domaine borné, et gardez `@lru_cache` avec un `maxsize` explicite si les arguments sont très variés.

## `@cached_property` pour les attributs calculés

`@cached_property` (Python 3.8+) applique le même principe à un attribut calculé : la méthode est exécutée à la première lecture, puis le résultat est conservé pour cette instance.

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

Le résultat est stocké directement dans l'instance (dans `doc.__dict__`), pas dans un cache partagé : il disparaît avec l'instance, sans risque de fuite mémoire. Pour forcer un nouveau calcul, on supprime l'attribut :

```python
del doc.nombre_mots  # force le recalcul au prochain accès
```

## Les arguments doivent être hashables

Le cache est un dictionnaire dont les clés sont construites à partir des arguments, qui doivent donc être hashables. Cela exclut les types modifiables comme `list`, `dict` ou `set` :

```python
@lru_cache
def somme(items):
    return sum(items)

somme((1, 2, 3))  # OK : tuple est hashable
somme([1, 2, 3])  # TypeError: unhashable type: 'list'
```

Pour contourner la limite, on convertit l'argument avant l'appel, en `tuple` en général, ou en `frozenset` si l'ordre et les doublons n'ont pas d'importance pour la fonction :

```python
ma_liste = [1, 2, 3, 2, 1]
total = somme(tuple(ma_liste))
print(total)  # 9
```

## Les méthodes d'instance

`@lru_cache` fonctionne sur une méthode, mais `self` fait alors partie des arguments, donc de la clé du cache :

```python
class Calculateur:
    @lru_cache
    def calcul(self, x):
        return x * 2
```

Le cache, partagé par toutes les instances de la classe, garde une référence vers chaque instance utilisée. Une instance ne peut donc pas être libérée tant que son entrée n'est pas sortie du cache ou que le cache n'a pas été vidé. Avec `@cache` ou `maxsize=None`, aucune entrée ne sort d'elle-même du cache : un programme qui crée beaucoup d'instances voit sa mémoire grossir.

Si la valeur ne dépend que de l'instance, `@cached_property` est plus adapté. Sinon, on peut sortir la méthode de la classe pour en faire une fonction qui reçoit seulement les attributs dont elle a besoin, et mettre le cache sur cette fonction.

## Les fonctions async

`@lru_cache` ne fonctionne pas avec une fonction `async` : il met en cache l'objet coroutine renvoyé par l'appel, pas son résultat. Or une coroutine ne peut être attendue qu'une seule fois. Le deuxième `await` avec le même argument lève donc `RuntimeError: cannot reuse already awaited coroutine`.

```python
@lru_cache
async def fetch(url):  # piège
    ...
```

Pour une fonction `async`, il faut passer par une bibliothèque dédiée comme `async-lru` ou `aiocache`.

## Le cache est local au processus

`@lru_cache` stocke ses entrées dans la mémoire du processus Python. Le cache est donc perdu à chaque redémarrage de l'application, et plusieurs processus, comme les *workers* de Gunicorn, ont chacun le leur. Pour un cache partagé entre processus ou qui survit aux redémarrages, il faut un stockage externe comme Redis : c'est ce que montrent les articles sur le cache avec Flask, Django et FastAPI, en lien plus bas.

Au sein d'un même processus, le cache est thread-safe : sa structure reste cohérente même si plusieurs threads l'utilisent en même temps. Par contre, si un thread appelle la fonction alors qu'un autre est encore en train de calculer le résultat pour les mêmes arguments, la fonction peut être exécutée une deuxième fois : le cache ne regroupe pas les appels en cours.

## Quelles fonctions mettre en cache

Le cache ne convient qu'aux fonctions dont le résultat dépend uniquement de leurs arguments. Une fonction qui a des effets de bord (écriture en base, envoi d'un email, modification d'un fichier) ne serait exécutée qu'au premier appel, et une fonction qui dépend de l'heure ou du hasard renverrait toujours la même valeur. Attention aussi aux fonctions qui renvoient un objet modifiable, une liste par exemple : c'est le même objet qui est renvoyé à chaque appel, et toute modification de cet objet change aussi la valeur en cache.

Voici trois situations où le cache est utile.

### Les algorithmes récursifs

C'est l'usage classique : un calcul récursif qui repasse sans cesse par les mêmes valeurs, comme le nombre de combinaisons de `k` éléments parmi `n`.

```python
from functools import cache

@cache
def combinaisons(n, k):
    if k == 0 or k == n:
        return 1
    return combinaisons(n - 1, k - 1) + combinaisons(n - 1, k)
```

Sans cache, `combinaisons(30, 15)` déclencherait plus de 300 millions d'appels. Avec `@cache`, chacun des 255 couples `(n, k)` rencontrés n'est calculé qu'une seule fois.

### Les appels réseau

Une fonction qui interroge une API gagne beaucoup à être mise en cache, à condition que les données ne changent pas pendant l'exécution du programme :

```python
import requests
from functools import lru_cache

@lru_cache(maxsize=512)
def fetch_user(user_id):
    response = requests.get(f"https://api.exemple.com/users/{user_id}")
    return response.json()
```

Si plusieurs parties du code demandent le même utilisateur, on évite des allers-retours réseau. Par contre, si les données peuvent changer en cours de route, le cache renverra une version périmée : il faut alors le vider au bon moment avec `cache_clear()`, ou utiliser un cache avec une durée de vie, comme `TTLCache` ou le décorateur `ttl_cache` de la bibliothèque `cachetools`.

### Les expressions régulières

Quand un objet coûteux à construire dépend d'une clé qui revient souvent, on peut mettre sa construction en cache :

```python
import re
from functools import lru_cache

@lru_cache
def compile_regex(pattern):
    return re.compile(pattern)
```

Pour les expressions régulières, le module `re` applique d'ailleurs déjà ce principe : `re.compile` et les fonctions comme `re.search` gardent en cache les derniers motifs compilés (jusqu'à 512 en Python 3.13, valeur de la constante privée `re._MAXCACHE`). Appeler `re.compile` plusieurs fois avec le même motif coûte donc peu. Un `@lru_cache` comme celui-ci n'apporte quelque chose que si vous utilisez plus de motifs différents que ce cache interne ne peut en garder, sachant qu'il est partagé avec tout le reste du programme.

## Voir aussi

- [Python : Comment utiliser les décorateurs]({% post_url 2026-05-14-Python-les-decorateurs %})
- [Comment ajouter un cache à une application Flask]({% post_url 2025-09-14-Comment-utiliser-un-cache-avec-Flask %})
- [Comment ajouter du cache à une application Django]({% post_url 2025-11-01-Comment-ajouter-du-cache-a-une-application-Django %})
- [Ajouter un cache à notre application FastAPI avec redis]({% post_url 2025-08-18-Utiliser-fastapi-cache2-avec-FastAPI %})
- [Documentation officielle de `functools`](https://docs.python.org/fr/3/library/functools.html)
- [FAQ Python : How do I cache method calls?](https://docs.python.org/3/faq/programming.html#how-do-i-cache-method-calls)
