---
layout: article
title: "Python : Comment tester son code avec pytest"
tags:
    - python
    - test
    - pytest
author: Pierre Chopinet
---

Tester son code est une étape essentielle du développement logiciel. Dans ce
tutoriel, vous allez apprendre à écrire et exécuter des tests en Python avec
pytest, le framework de test le plus populaire de l'écosystème Python.
<!--more-->
pytest se distingue par sa simplicité d'utilisation : pas besoin de classes,
pas de boilerplate, il suffit d'écrire des fonctions dont le nom commence par
`test_` et d'utiliser le mot-clé `assert` de Python. C'est aussi un outil
très puissant qui propose des fixtures, du paramétrage de tests, et un
écosystème de plugins très riche.

L'objectif de ce tutoriel est d'apprendre comment :

- Écrire et exécuter des tests avec pytest
- Organiser ses fichiers de test dans un projet
- Utiliser les fixtures pour préparer des données de test
- Paramétrer ses tests pour couvrir plusieurs cas
- Vérifier qu'une fonction lève bien une exception

## Installation

Pour commencer, il vous faut un interpréteur python en version 3. Ensuite,
installez pytest avec pip :

```bash
pip3 install pytest
```


## Pourquoi écrire des tests ?

Quand on écrit du code, on le teste souvent "à la main" : on lance le programme,
on vérifie que le résultat est correct, et on passe à la suite. Le problème,
c'est que cette vérification manuelle ne passe pas à l'échelle. Dès que le
projet grandit, on ne peut plus tout revérifier à chaque modification.

Les tests automatisés permettent de s'assurer que le code fonctionne comme
prévu à chaque changement. Ils servent de filet de sécurité : si une
modification casse quelque chose, les tests le détectent immédiatement. C'est
particulièrement utile quand on travaille en équipe ou qu'on revient sur du code
écrit il y a plusieurs mois.

## Un premier test

Commençons par un exemple simple. Imaginons qu'on a un fichier `calcul.py` avec
une fonction d'addition :

```python
def addition(a, b):
    return a + b
```

Pour tester cette fonction, on crée un fichier `test_calcul.py` :

```python
from calcul import addition

def test_addition():
    assert addition(1, 2) == 3
```

C'est tout. Pas de classe à hériter, pas de méthode spéciale à appeler. On
importe la fonction, on l'appelle, et on vérifie le résultat avec `assert`.

Pour lancer le test :

```bash
pytest
```

pytest va automatiquement découvrir tous les fichiers dont le nom commence par
`test_` et exécuter toutes les fonctions dont le nom commence aussi par `test_`.

Le résultat devrait ressembler à :

```text
========================= test session starts ==========================
collected 1 item

test_calcul.py .                                                 [100%]

========================== 1 passed in 0.01s ===========================
```

Le point `.` signifie que le test est passé. Si le test échoue, pytest affiche
un `F` et un message d'erreur détaillé.

## Comprendre les messages d'erreur

Modifions notre fonction pour qu'elle soit volontairement incorrecte :

```python
def addition(a, b):
    return a * b  # bug volontaire
```

En relançant `pytest`, on obtient :

```text
========================= test session starts ==========================
collected 1 item

test_calcul.py F                                                 [100%]

=============================== FAILURES ===============================
__________________________ test_addition ___________________________

    def test_addition():
>       assert addition(1, 2) == 3
E       assert 2 == 3
E        +  where 2 = addition(1, 2)

test_calcul.py:4: AssertionError
======================== 1 failed in 0.01s =============================
```

pytest montre exactement quelle assertion a échoué, quelle valeur a été obtenue
(`2`) et quelle valeur était attendue (`3`). C'est l'un des gros avantages de
pytest par rapport au module `unittest` de la bibliothèque standard : les
messages d'erreur sont beaucoup plus lisibles.

> Pour avoir encore plus de détails, lancez pytest avec l'option `-v`
> (verbose) : `pytest -v`

## Organiser ses tests

Sur un vrai projet, on ne met pas ses tests à côté de son code source. La
convention la plus courante est de créer un dossier `tests/` à la racine du
projet :

```text
mon_projet/
    mon_projet/
        __init__.py
        calcul.py
        utils.py
    tests/
        __init__.py
        test_calcul.py
        test_utils.py
```

Le fichier `__init__.py` dans `tests/` peut être vide, il sert à indiquer à
Python que c'est un package. Avec cette structure, on lance toujours les tests
depuis la racine du projet avec `pytest`, et il découvrira tout seul les
fichiers dans `tests/`.

On peut aussi lancer un seul fichier de test :

```bash
pytest tests/test_calcul.py
```

Ou même un seul test spécifique :

```bash
pytest tests/test_calcul.py::test_addition
```

## Les fixtures

Quand plusieurs tests ont besoin des mêmes données ou de la même mise en place,
on utilise des **fixtures**. Une fixture est une fonction décorée avec
`@pytest.fixture` qui prépare quelque chose pour les tests.

Prenons un exemple concret. Imaginons une classe `Panier` qui gère un panier
d'achat :

```python
class Panier:
    def __init__(self):
        self.articles = []

    def ajouter(self, nom, prix):
        self.articles.append({"nom": nom, "prix": prix})

    def total(self):
        return sum(a["prix"] for a in self.articles)

    def nombre_articles(self):
        return len(self.articles)
```

Plutôt que de créer un panier dans chaque test, on peut utiliser une fixture :

```python
import pytest
from panier import Panier

@pytest.fixture
def panier_avec_articles():
    p = Panier()
    p.ajouter("Clavier", 49.99)
    p.ajouter("Souris", 29.99)
    return p

def test_total(panier_avec_articles):
    assert panier_avec_articles.total() == 79.98

def test_nombre_articles(panier_avec_articles):
    assert panier_avec_articles.nombre_articles() == 2
```

Pour utiliser la fixture, il suffit de mettre son nom en paramètre de la
fonction de test. pytest se charge d'appeler la fixture et de passer le résultat
au test. Chaque test reçoit une **nouvelle instance** du panier : les tests sont
donc complètement indépendants les uns des autres.

> Les fixtures peuvent aussi utiliser d'autres fixtures en paramètre, ce qui
> permet de composer des setups complexes de manière lisible.

## Paramétrer ses tests

Quand on veut tester une fonction avec plusieurs jeux de données, on pourrait
écrire un test par cas. Mais pytest propose une solution bien plus élégante
avec `@pytest.mark.parametrize` :

```python
import pytest
from calcul import addition

@pytest.mark.parametrize("a, b, attendu", [
    (1, 2, 3),
    (0, 0, 0),
    (-1, 1, 0),
    (100, 200, 300),
    (0.1, 0.2, pytest.approx(0.3)),
])
def test_addition(a, b, attendu):
    assert addition(a, b) == attendu
```

pytest va générer un test pour chaque tuple de la liste. En lançant `pytest -v`,
on voit bien les différents cas :

```text
test_calcul.py::test_addition[1-2-3] PASSED
test_calcul.py::test_addition[0-0-0] PASSED
test_calcul.py::test_addition[-1-1-0] PASSED
test_calcul.py::test_addition[100-200-300] PASSED
test_calcul.py::test_addition[0.1-0.2-0.3] PASSED
```

> Notez l'utilisation de `pytest.approx(0.3)` pour la comparaison de
> flottants. En raison de la représentation en virgule flottante,
> `0.1 + 0.2` ne donne pas exactement `0.3` en Python. `pytest.approx`
> permet de comparer avec une tolérance par défaut.

## Tester les exceptions

Parfois, on veut vérifier qu'une fonction lève bien une exception dans certains
cas. Par exemple, si on a une fonction de division :

```python
def division(a, b):
    if b == 0:
        raise ValueError("Division par zéro impossible")
    return a / b
```

On peut tester que l'exception est bien levée avec `pytest.raises` :

```python
import pytest
from calcul import division

def test_division_par_zero():
    with pytest.raises(ValueError, match="Division par zéro"):
        division(10, 0)

def test_division_normale():
    assert division(10, 2) == 5.0
```

Le `with pytest.raises(ValueError)` vérifie que le bloc lève bien une
`ValueError`. Le paramètre `match` est optionnel et permet de vérifier que le
message de l'exception correspond au pattern donné (c'est une expression
régulière).

## Fichier conftest.py

Quand on a des fixtures utilisées par plusieurs fichiers de test, on peut les
placer dans un fichier spécial appelé `conftest.py`. pytest le découvre
automatiquement et rend les fixtures disponibles pour tous les tests du même
dossier (et ses sous-dossiers).

```text
tests/
    conftest.py
    test_calcul.py
    test_panier.py
```

```python
# tests/conftest.py
import pytest
from panier import Panier

@pytest.fixture
def panier_vide():
    return Panier()

@pytest.fixture
def panier_avec_articles():
    p = Panier()
    p.ajouter("Clavier", 49.99)
    p.ajouter("Souris", 29.99)
    return p
```

Les fixtures définies dans `conftest.py` sont alors utilisables dans
`test_calcul.py` et `test_panier.py` sans avoir besoin de les importer. C'est
la manière recommandée de partager des fixtures entre fichiers de test.

## Quelques options utiles

pytest propose de nombreuses options en ligne de commande. Voici les plus
courantes :

```bash
# Lancer les tests avec un affichage détaillé
pytest -v

# Arrêter dès le premier échec
pytest -x

# Afficher les print() dans la sortie
pytest -s

# Lancer uniquement les tests qui contiennent "panier" dans leur nom
pytest -k "panier"

# Combiner les options
pytest -v -x -s
```

L'option `-k` est particulièrement pratique pour lancer un sous-ensemble de
tests sans avoir à spécifier les chemins exacts.

## Voir aussi

- [Python : Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %})
- [Python : Comment faire une api web avec Flask]({% post_url 2021-04-20-Comment-faire-une-api-web-en-python %})
- [Python : Comment faire des requêtes HTTP avec requests]({% post_url 2020-05-22-Comment-faire-des-requetes-http-en-python-avec-requests %})
- [Documentation officielle de pytest](https://docs.pytest.org/)
