---
layout: article
title: "Python : Comment tester son code avec pytest"
tags:
    - python
    - test
    - pytest
author: Pierre Chopinet
---

Des tests automatisés permettent de vérifier que son code fait ce qu'on attend,
et qu'il continue à le faire après chaque modification. Dans ce tutoriel, vous
allez apprendre à écrire et exécuter des tests en Python avec pytest, le
framework de test le plus populaire de l'écosystème Python.
<!--more-->

pytest se distingue par sa simplicité d'utilisation : pas besoin de classes,
pas de boilerplate, il suffit d'écrire des fonctions dont le nom commence par
`test_` et d'utiliser le mot-clé `assert` de Python. Il propose aussi des
fixtures, le paramétrage des tests et de nombreux plugins.

L'objectif de ce tutoriel est d'apprendre comment :

- Écrire et exécuter des tests avec pytest
- Organiser ses fichiers de test dans un projet
- Utiliser les fixtures pour préparer des données de test
- Paramétrer ses tests pour couvrir plusieurs cas
- Vérifier qu'une fonction lève bien une exception

## Installation

Pour commencer, il vous faut Python. Ensuite, installez pytest avec pip :

```bash
pip3 install pytest
```

Les exemples de cet article ont été testés avec Python 3.13 et pytest 9.1.

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

Le résultat devrait ressembler à ceci (sans les lignes `platform` et `rootdir`
de l'en-tête, qui dépendent de votre machine) :

```text
============================= test session starts ==============================
collected 1 item

test_calcul.py .                                                         [100%]

============================== 1 passed in 0.00s ===============================
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
============================= test session starts ==============================
collected 1 item

test_calcul.py F                                                         [100%]

=================================== FAILURES ===================================
________________________________ test_addition _________________________________

    def test_addition():
>       assert addition(1, 2) == 3
E       assert 2 == 3
E        +  where 2 = addition(1, 2)

test_calcul.py:4: AssertionError
=========================== short test summary info ============================
FAILED test_calcul.py::test_addition - assert 2 == 3
============================== 1 failed in 0.01s ===============================
```

pytest montre exactement quelle assertion a échoué, quelle valeur a été obtenue
(`2`) et quelle valeur était attendue (`3`). C'est l'un des gros avantages de
pytest par rapport au module `unittest` de la bibliothèque standard : les
messages d'erreur sont beaucoup plus lisibles. La section `short test summary
info`, à la fin, liste les tests en échec, ce qui est pratique quand il y en a
beaucoup.

Pour avoir plus de détails, par exemple le nom et le résultat de chaque test,
lancez pytest avec l'option `-v` (*verbose*) : `pytest -v`.

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

Le fichier `__init__.py` dans `tests/` peut être vide. Il fait de `tests` un
package, et pytest ajoute alors la racine du projet au chemin d'import : les
tests peuvent importer le code avec `from mon_projet.calcul import addition`.
Sans ce fichier, cet import échoue avec une `ModuleNotFoundError` quand on lance
`pytest` (mais pas avec `python -m pytest`, qui ajoute le dossier courant au
chemin d'import). Avec cette structure, on lance toujours les tests depuis la
racine du projet, et pytest découvre tout seul les fichiers dans `tests/`.

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
on utilise des *fixtures*. Une fixture est une fonction décorée avec
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
au test. Chaque test reçoit une nouvelle instance du panier : les tests sont
donc complètement indépendants les uns des autres.

Une fixture peut elle-même utiliser d'autres fixtures, en les déclarant en
paramètre de la même façon. On compose ainsi des mises en place plus complexes
à partir de fixtures simples.

## Paramétrer ses tests

Quand on veut tester une fonction avec plusieurs jeux de données, on pourrait
écrire un test par cas. pytest permet de faire plus court avec
`@pytest.mark.parametrize` :

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
test_calcul.py::test_addition[1-2-3] PASSED                              [ 20%]
test_calcul.py::test_addition[0-0-0] PASSED                              [ 40%]
test_calcul.py::test_addition[-1-1-0] PASSED                             [ 60%]
test_calcul.py::test_addition[100-200-300] PASSED                        [ 80%]
test_calcul.py::test_addition[0.1-0.2-attendu4] PASSED                   [100%]
```

Notez l'utilisation de `pytest.approx(0.3)` pour le dernier cas. En raison de
la représentation des nombres en virgule flottante, `0.1 + 0.2` ne donne pas
exactement `0.3` en Python, mais `0.30000000000000004`. `pytest.approx` compare
avec une petite tolérance. C'est aussi pour ça que le dernier identifiant
affiche `attendu4` : pour un objet comme `approx`, pytest construit l'identifiant
à partir du nom du paramètre et de l'indice du cas, plutôt qu'à partir de la
valeur.

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

# Lancer uniquement les tests dont le nom (ou celui du fichier) contient "panier"
pytest -k "panier"

# Combiner les options
pytest -v -x -s
```

L'option `-k` est particulièrement pratique pour lancer un sous-ensemble de
tests sans avoir à spécifier les chemins exacts.

## Voir aussi

- [Python : Comment utiliser les décorateurs]({% post_url 2026-05-14-Python-les-decorateurs %})
- [Python : Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %})
- [Python : Comment faire une api web avec Flask]({% post_url 2021-04-20-Comment-faire-une-api-web-en-python %})
- [Documentation officielle de pytest](https://docs.pytest.org/)
