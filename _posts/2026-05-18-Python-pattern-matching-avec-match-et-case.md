---
layout: article
title: "Python : Le pattern matching avec match et case"
tags:
  - python
  - pattern-matching
  - match-case
author: Pierre Chopinet
---

Python 3.10 (octobre 2021) a introduit l'instruction `match`, qui ressemble au `switch` d'autres langages mais va plus loin : en plus de comparer des valeurs, elle sait déstructurer des listes, des dictionnaires ou des objets, et en extraire les champs dans la même ligne.
<!--more-->

Si vous avez déjà croisé le pattern matching en [Java]({% post_url 2025-10-23-Pattern-matching-en-Java-moderne %}), en Rust ou en OCaml, c'est la même idée : remplacer des chaînes de `if/elif` par des motifs qui décrivent la forme des données attendues.

Dans cet article :
- Pourquoi match / case ?
- Comparer des valeurs
- Capturer une valeur
- Déstructurer une liste ou un tuple
- Déstructurer un dictionnaire
- Déstructurer un objet
- Imbriquer les motifs
- Ajouter une condition avec `if`
- Un nom simple est toujours une capture
- Router des messages
- Un mini-évaluateur d'expressions
- Quand préférer un `if`

Pré-requis : Python 3.10 ou plus récent. Les exemples ont été testés avec Python 3.13.

## Pourquoi match / case ?

Sans pattern matching, on enchaîne souvent des `if/elif` qui mélangent vérification de type, accès aux clés et extraction des valeurs :

```python
def decrire(message):
    if isinstance(message, dict) and message.get("type") == "ping":
        return "pong"
    elif isinstance(message, dict) and message.get("type") == "echo":
        return message.get("data", "")
    elif isinstance(message, dict) and message.get("type") == "error":
        code = message.get("code")
        msg = message.get("message", "")
        return f"Erreur {code} : {msg}"
    else:
        return "message inconnu"
```

Avec `match`, on écrit :

```python
def decrire(message):
    match message:
        case {"type": "ping"}:
            return "pong"
        case {"type": "echo", "data": data}:
            return data
        case {"type": "error", "code": code, "message": msg}:
            return f"Erreur {code} : {msg}"
        case _:
            return "message inconnu"
```

Chaque `case` décrit la forme du message attendu et extrait les champs dont on a besoin, sans `.get()` ni variables intermédiaires. Seule différence avec la première version : un message `echo` sans champ `data` tombe maintenant dans le cas par défaut, puisque le motif exige la présence de la clé.

## Comparer des valeurs

Le cas le plus simple consiste à comparer une valeur à des littéraux :

```python
def description_statut(code):
    match code:
        case 200:
            return "OK"
        case 404:
            return "Not Found"
        case 500:
            return "Internal Server Error"
        case _:
            return "Statut inconnu"

print(description_statut(200))  # OK
print(description_statut(418))  # Statut inconnu
```

Les `case` sont testés dans l'ordre, et le premier qui correspond l'emporte. Le motif `_` correspond à tout, c'est l'équivalent du `default` d'un `switch`. Sans lui, un `match` qui ne trouve aucun `case` correspondant ne fait rien, sans lever d'exception : prévoyez donc un `case _` dès qu'une valeur inattendue est possible.

Pour accepter plusieurs valeurs dans un même `case`, on les sépare par `|` :

```python
def categorie_http(code):
    match code:
        case 200 | 201 | 204:
            return "succès"
        case 301 | 302 | 308:
            return "redirection"
        case 400 | 401 | 403 | 404 | 422:
            return "erreur client"
        case 500 | 502 | 503 | 504:
            return "erreur serveur"
        case _:
            return "autre"

print(categorie_http(404))  # erreur client
```

Notez que `match` est une instruction et non une expression : contrairement aux expressions `switch` de Java ou au `match` de Rust, il ne renvoie pas de valeur. On ne peut donc pas écrire `resultat = match x: ...`, d'où le `return` dans chaque `case` et le `match` placé dans une fonction, comme dans tous les exemples de cet article.

## Capturer une valeur

Un `case` peut aussi capturer la valeur dans une variable :

```python
def describe(value):
    match value:
        case 0:
            return "zéro"
        case n:
            return f"nombre {n}"

print(describe(0))    # zéro
print(describe(42))   # nombre 42
```

`case n` correspond à n'importe quelle valeur et la range dans la variable `n`, utilisable dans le corps du `case`. Ce comportement est à l'origine du piège le plus courant avec `match`, détaillé plus bas.

## Déstructurer une liste ou un tuple

Les motifs de séquence vérifient le nombre d'éléments et les extraient :

```python
def analyser_commande(tokens):
    match tokens:
        case []:
            return "commande vide"
        case [cmd]:
            return f"commande sans argument : {cmd}"
        case [cmd, arg]:
            return f"{cmd} avec un argument : {arg}"
        case [cmd, *args]:
            return f"{cmd} avec {len(args)} arguments : {args}"

print(analyser_commande([]))                   # commande vide
print(analyser_commande(["ls"]))               # commande sans argument : ls
print(analyser_commande(["cp", "a", "b"]))     # cp avec 2 arguments : ['a', 'b']
```

`[cmd, arg]` ne correspond qu'à une séquence de deux éléments. Avec `*`, on capture le reste : `[cmd, *args]` met le premier élément dans `cmd` et les suivants dans la liste `args`. On peut aussi récupérer le milieu avec `[premier, *milieu, dernier]`.

Attention, une chaîne de caractères ne correspond jamais à un motif de séquence, même si elle est itérable : `case [a, b]` ne correspond pas à `"ab"`. C'est voulu. Pour traiter une chaîne, on utilise le motif de type `case str()`, puis on la manipule dans le corps du `case`.

## Déstructurer un dictionnaire

Un motif de dictionnaire vérifie que certaines clés sont présentes, avec les bonnes valeurs :

```python
def traiter_evenement(event):
    match event:
        case {"type": "click", "x": x, "y": y}:
            return f"clic en ({x}, {y})"
        case {"type": "keypress", "key": key}:
            return f"touche pressée : {key}"
        case {"type": "scroll", "delta": delta}:
            return f"scroll de {delta}"
        case _:
            return "événement non géré"

print(traiter_evenement({"type": "click", "x": 10, "y": 20, "timestamp": 1234}))
# clic en (10, 20)
```

Les clés qui ne sont pas dans le motif, comme `timestamp` ici, sont ignorées. C'est l'inverse des motifs de séquence, qui imposent la longueur exacte. Pour récupérer les clés restantes, on utilise `**` :

```python
def resumer(event):
    match event:
        case {"type": type_, **autres_champs}:
            return f"type={type_}, autres={autres_champs}"

print(resumer({"type": "click", "x": 10, "y": 20}))
# type=click, autres={'x': 10, 'y': 20}
```

## Déstructurer un objet

On peut aussi vérifier le type d'un objet et extraire ses attributs dans le même motif. Le plus simple est de partir d'une `dataclass` :

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: float
    y: float

def quadrant(p):
    match p:
        case Point(0, 0):
            return "origine"
        case Point(x=0, y=_):
            return "axe Y"
        case Point(x=_, y=0):
            return "axe X"
        case Point(x, y) if x > 0 and y > 0:
            return "Q1"
        case Point(x, y) if x < 0 and y > 0:
            return "Q2"
        case Point(x, y) if x < 0 and y < 0:
            return "Q3"
        case Point(x, y) if x > 0 and y < 0:
            return "Q4"

print(quadrant(Point(0, 0)))    # origine
print(quadrant(Point(3, 4)))    # Q1
print(quadrant(Point(-2, -5)))  # Q3
```

Deux syntaxes sont possibles. La forme positionnelle, `Point(0, 0)`, s'appuie sur l'attribut `__match_args__` que `@dataclass` génère à partir de l'ordre des champs. La forme par mot-clé, `Point(x=0, y=_)`, est plus explicite et continue de fonctionner si l'ordre des attributs change : c'est celle à privilégier pour une classe qui a plus de deux ou trois attributs.

Avec une classe classique, la forme par mot-clé fonctionne directement. Pour la forme positionnelle, il faut définir `__match_args__` soi-même, sinon Python lève une erreur `TypeError: Utilisateur() accepts 0 positional sub-patterns (2 given)` :

```python
class Utilisateur:
    __match_args__ = ("nom", "role")

    def __init__(self, nom, role):
        self.nom = nom
        self.role = role

def saluer(u):
    match u:
        case Utilisateur(nom, "admin"):
            return f"Bonjour {nom} (admin)"
        case Utilisateur(nom, _):
            return f"Bonjour {nom}"

print(saluer(Utilisateur("Alice", "admin")))   # Bonjour Alice (admin)
print(saluer(Utilisateur("Bob", "viewer")))    # Bonjour Bob
```

## Imbriquer les motifs

Les motifs se combinent entre eux : un objet peut contenir d'autres objets, une liste des dictionnaires, etc.

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: float
    y: float

@dataclass
class Segment:
    debut: Point
    fin: Point

def longueur_manhattan(s):
    match s:
        case Segment(Point(x1, y1), Point(x2, y2)):
            return abs(x1 - x2) + abs(y1 - y2)

print(longueur_manhattan(Segment(Point(0, 0), Point(3, 4))))  # 7
```

C'est ce qui rend `match` pratique pour parcourir des structures arborescentes : arbres syntaxiques, documents JSON, fichiers de configuration.

## Ajouter une condition avec `if`

Un `case` peut être complété par une condition `if`, appelée *guard* :

```python
def classer(valeur):
    match valeur:
        case int(n) if n < 0:
            return "entier négatif"
        case int(n) if n == 0:
            return "zéro"
        case int(n) if n > 0:
            return "entier positif"
        case float():
            return "nombre flottant"
        case _:
            return "autre type"

print(classer(-5))    # entier négatif
print(classer(0))     # zéro
print(classer(3.14))  # nombre flottant
```

`int(n)` est un motif de type : il correspond si la valeur est un `int` et la capture dans `n`. La condition n'est évaluée que si le motif correspond, et si elle est fausse, Python passe au `case` suivant.

Attention, `bool` est une sous-classe d'`int` : `classer(True)` correspond à `case int(n)` (avec `n` qui vaut `True`, soit 1) et renvoie donc `"entier positif"`. Pour distinguer les booléens des entiers, il faut placer un `case bool()` avant les `case int()`.

## Un nom simple est toujours une capture

Voici l'erreur la plus fréquente avec `match`. On veut comparer un statut à des constantes :

```python
ACTIF = "actif"
INACTIF = "inactif"

def message_statut(s):
    match s:
        case ACTIF:      # piège ! ce n'est PAS une comparaison
            return "utilisateur actif"
        case INACTIF:
            return "utilisateur inactif"
```

`case ACTIF` n'est pas une comparaison avec la constante, mais une capture comme le `case n` vu plus haut : n'importe quelle valeur correspond et se retrouve dans une nouvelle variable `ACTIF`. Dans cet exemple, Python s'en rend compte, puisque le second `case` ne pourrait jamais être atteint, et refuse de compiler la fonction :

```
SyntaxError: name capture 'ACTIF' makes remaining patterns unreachable
```

Le piège est plus sournois quand la capture se trouve dans le dernier `case`, ou dans le seul : aucune erreur, et la fonction répond "utilisateur actif" quel que soit le statut. En dehors d'une fonction, la capture écrase même la constante, car les variables capturées restent définies après le `match` :

```python
ACTIF = "actif"

match "inactif":
    case ACTIF:
        pass

print(ACTIF)  # inactif
```

Pour comparer à une constante, il faut un nom qualifié, avec un point : `Classe.ATTRIBUT`, `module.CONSTANTE`. On peut regrouper les constantes dans une classe :

```python
class Statut:
    ACTIF = "actif"
    INACTIF = "inactif"

def message_statut(s):
    match s:
        case Statut.ACTIF:     # OK : le point en fait une comparaison
            return "utilisateur actif"
        case Statut.INACTIF:
            return "utilisateur inactif"
        case _:
            return "statut inconnu"
```

Ou, plus naturellement, utiliser une Enum :

```python
from enum import Enum

class Statut(Enum):
    ACTIF = "actif"
    INACTIF = "inactif"

def message_statut(s):
    match s:
        case Statut.ACTIF:
            return "utilisateur actif"
        case Statut.INACTIF:
            return "utilisateur inactif"
        case _:
            return "statut inconnu"
```

Avec une Enum classique, la chaîne `"actif"` ne correspond pas à `Statut.ACTIF` : `message_statut("actif")` renvoie `"statut inconnu"`. Il faut convertir la valeur avant le `match` avec `Statut("actif")`, ou hériter de `StrEnum` (Python 3.11 et plus), dont les membres sont égaux à leur valeur.

## Router des messages

`match` est à l'aise pour aiguiller des messages structurés, ceux d'une API, d'un bot ou d'une CLI :

```python
def traiter(message):
    match message:
        case {"action": "create", "resource": "user", "data": {"name": nom}}:
            return f"Création de l'utilisateur {nom}"
        case {"action": "delete", "resource": "user", "id": user_id}:
            return f"Suppression de l'utilisateur {user_id}"
        case {"action": "list", "resource": resource}:
            return f"Liste des {resource}"
        case {"action": action}:
            return f"Action non gérée : {action}"
        case _:
            return "Message invalide"

print(traiter({"action": "create", "resource": "user", "data": {"name": "Alice"}}))
# Création de l'utilisateur Alice

print(traiter({"action": "delete", "resource": "user", "id": 42}))
# Suppression de l'utilisateur 42
```

Le premier motif va chercher le nom dans un dictionnaire imbriqué (`"data": {"name": nom}`). L'ordre des `case` compte : `{"action": action}` attrape toutes les actions qui n'ont pas été traitées avant lui, et `case _` tout ce qui n'a même pas de clé `action`.

## Un mini-évaluateur d'expressions

Dernier exemple, un évaluateur d'expressions arithmétiques représentées sous forme d'arbre. Chaque `case` traite un type de nœud, la récursion fait le reste :

```python
from dataclasses import dataclass

@dataclass
class Nombre:
    valeur: float

@dataclass
class Addition:
    gauche: object
    droite: object

@dataclass
class Multiplication:
    gauche: object
    droite: object

def evaluer(expr):
    match expr:
        case Nombre(v):
            return v
        case Addition(g, d):
            return evaluer(g) + evaluer(d)
        case Multiplication(g, d):
            return evaluer(g) * evaluer(d)

# (2 + 3) * 4
expression = Multiplication(
    Addition(Nombre(2), Nombre(3)),
    Nombre(4),
)
print(evaluer(expression))  # 20
```

## Quand préférer un `if`

`match` est intéressant quand il faut à la fois vérifier la forme des données et en extraire des valeurs : documents JSON, messages, arbres, hiérarchies de dataclasses ou d'Enum. Pour comparer une variable à deux ou trois valeurs, un `if/elif` reste plus lisible, de même quand les conditions sont des expressions qui ne rentrent pas dans un motif. Si vos `case` se contentent d'appeler une méthode différente selon le type de l'objet, le polymorphisme classique (une méthode redéfinie dans chaque classe) est souvent plus adapté. Enfin, `match` n'existe pas avant Python 3.10 : à éviter si votre code doit tourner sur une version plus ancienne.

## Voir aussi

- [Pattern matching en Java moderne]({% post_url 2025-10-23-Pattern-matching-en-Java-moderne %})
- [Les Sealed classes en Java]({% post_url 2026-01-14-Sealed-classes-en-Java %})
- [Records en Java : simplifier vos DTOs]({% post_url 2026-01-10-Records-en-Java-simplifier-vos-DTOs %})
- [PEP 634 - Structural Pattern Matching: Specification](https://peps.python.org/pep-0634/)
- [PEP 635 - Structural Pattern Matching: Motivation and Rationale](https://peps.python.org/pep-0635/)
- [PEP 636 - Structural Pattern Matching: Tutorial](https://peps.python.org/pep-0636/)
- [Documentation officielle : l'instruction match](https://docs.python.org/3/reference/compound_stmts.html#the-match-statement)
