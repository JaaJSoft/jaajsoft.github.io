---
layout: article
title: "Python : Le pattern matching avec match et case"
tags:
  - python
  - pattern-matching
  - match-case
author: Pierre Chopinet
---

Arrivé avec Python 3.10 (octobre 2021), le pattern matching va bien plus loin qu'un simple `switch / case` comme on en trouve dans d'autres langages. Il permet non seulement de comparer des valeurs, mais aussi de **déstructurer** des objets, des listes ou des dictionnaires en une seule expression.
<!--more-->

Si vous avez déjà croisé le pattern matching en [Java]({% post_url 2025-10-23-Pattern-matching-en-Java-moderne %}), en Rust ou en OCaml, c'est la même idée : remplacer des chaînes `if/elif/else` verbeuses par une syntaxe déclarative qui exprime à la fois la condition et l'extraction des données dans la même ligne.

Dans cet article :
- Le principe du `match` / `case` et quand l'utiliser
- Match sur des littéraux, des séquences, des dictionnaires
- Déstructurer des classes et des dataclasses
- Ajouter des conditions supplémentaires avec `if` (guards)
- Le piège classique du capture vs comparaison
- Bonnes pratiques et cas d'usage

Pré-requis : Python 3.10 ou plus récent.

---

## Pourquoi match / case ?

Sans pattern matching, on enchaîne souvent des `if/elif` qui mélangent vérification de type, accès à des attributs et extraction de valeurs :

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

Avec `match / case`, le même code devient :

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

La structure du message et l'extraction des champs apparaissent directement dans le `case`. Plus de `.get()`, plus de variables intermédiaires, plus de répétition.

---

## Premier exemple : match sur des littéraux

Le cas le plus simple : comparer une valeur à des littéraux.

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

Quelques règles :
- Les `case` sont testés dans l'ordre, le premier qui correspond gagne.
- Le wildcard `_` correspond à tout (équivalent au `default` d'un `switch`).
- Sans `case _`, un `match` qui ne correspond à rien ne fait simplement rien (pas d'exception).

---

## Capture de variables

Un `case` peut capturer la valeur dans une variable :

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

Ici, `case n` capture n'importe quelle valeur dans `n`, qui devient utilisable dans le corps du `case`. Attention, ce comportement est à la source du **piège le plus fréquent** de `match / case`, qu'on verra plus loin.

---

## Alternatives avec `|`

Pour regrouper plusieurs valeurs dans un même `case`, on utilise `|` :

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
```

---

## Match sur des séquences

Le pattern matching permet de déstructurer des listes et des tuples :

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

- `[cmd, *args]` capture le premier élément dans `cmd` et le reste dans `args`.
- On peut aussi capturer le milieu : `[premier, *milieu, dernier]`.

> Les chaînes de caractères **ne correspondent pas** aux patterns de séquence, même si elles sont itérables. C'est volontaire : `case [a, b]` ne matchera pas la chaîne `"ab"`. Pour matcher des caractères, faites un `case str()` puis traitez la chaîne.

---

## Match sur des dictionnaires

Les patterns de dictionnaire vérifient que **certaines clés** sont présentes avec les bonnes valeurs :

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

Point important : le pattern `{"type": "click", "x": x, "y": y}` **n'exige pas** que ce soient les seules clés. Les clés supplémentaires (comme `timestamp` ci-dessus) sont autorisées. C'est le comportement opposé des patterns de séquence, qui sont stricts sur la longueur.

Pour capturer toutes les clés restantes, utilisez `**rest` :

```python
match event:
    case {"type": type_, **autres_champs}:
        return f"type={type_}, autres={autres_champs}"
```

---

## Match sur des classes

On peut matcher sur le type d'un objet et déstructurer ses attributs en même temps. Le plus simple est d'utiliser une `dataclass`, qui fournit automatiquement les informations nécessaires :

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

Deux syntaxes coexistent dans le pattern :
- **Positionnelle** : `Point(0, 0)` matche un `Point` avec `x=0` et `y=0`. Fonctionne grâce à `__match_args__` (fourni automatiquement par `@dataclass`).
- **Par mot-clé** : `Point(x=0, y=_)` est explicite et plus robuste si l'ordre des attributs change.

### Sans dataclass

Pour une classe normale, il faut définir `__match_args__` pour activer la forme positionnelle :

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

---

## Combinaison : match imbriqué

Les patterns se composent. On peut matcher une liste de dictionnaires, ou un dataclass qui contient d'autres dataclasses :

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

Cette capacité à imbriquer rend `match / case` particulièrement bien adapté au traitement de structures arborescentes (AST, JSON, configuration).

---

## Guards : conditions supplémentaires avec `if`

Un `case` peut être complété par une condition `if` (appelée *guard*) :

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

`int(n)` est un pattern de type : il matche si la valeur est un `int`, et capture la valeur dans `n`. Le guard `if n < 0` ajoute une contrainte supplémentaire.

---

## Le piège du capture vs comparaison

Voici l'erreur la plus fréquente avec `match / case`. Imaginons qu'on veuille tester un statut contre des constantes :

```python
ACTIF = "actif"
INACTIF = "inactif"

def message_statut(s):
    match s:
        case ACTIF:      # piège ! ce n'est PAS une comparaison
            return "utilisateur actif"
        case INACTIF:    # inatteignable
            return "utilisateur inactif"
```

Ce code ne fonctionne pas : `case ACTIF` est interprété comme une **capture**, comme `case n` plus haut. Python crée une nouvelle variable nommée `ACTIF` qui capture la valeur de `s`, peu importe sa valeur. Le `case` matche toujours, et le deuxième est inatteignable.

La règle : **un nom simple (`ACTIF`) est toujours une capture, jamais une comparaison.**

Pour comparer à une constante, deux solutions :

### Solution 1 : qualifier le nom avec un point

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

### Solution 2 : utiliser une Enum (recommandé)

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

> Retenez : si le nom contient un point (`Module.CONSTANTE`, `Classe.attribut`, `enum.VALEUR`), c'est une comparaison. Sinon, c'est une capture.

---

## Match n'est pas une expression

Contrairement au `switch` de Java 21 ou au `match` de Rust, le `match` Python est une **instruction**, pas une expression. Il ne retourne pas directement de valeur :

```python
# Ne fonctionne pas :
# resultat = match x:
#     case 0:
#         "zéro"
#     case _:
#         "autre"

# La forme correcte :
def description(x):
    match x:
        case 0:
            return "zéro"
        case _:
            return "autre"
```

Cette limitation rend souvent utile d'encapsuler le `match` dans une fonction qui retourne la valeur calculée, comme dans tous les exemples ci-dessus.

---

## Cas d'usage : router des messages

Un cas où `match / case` brille : router des messages typés dans une application réseau, un bot, ou une CLI.

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

Sans `match`, ce code demanderait une cascade de `if/elif` avec beaucoup de `.get()` et de variables intermédiaires.

---

## Cas d'usage : parser un AST simple

Le pattern matching imbriqué est particulièrement adapté aux structures arborescentes. Voici un mini-évaluateur d'expressions arithmétiques :

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

Ce style déclaratif rend la logique d'évaluation directement lisible.

---

## Bonnes pratiques

### À faire

- **Utiliser `match / case` pour la déstructuration**, pas pour une simple comparaison à 2-3 valeurs (où un `if/elif` reste plus lisible).
- **Toujours prévoir un `case _:`** pour les cas non gérés, sinon le `match` peut passer silencieusement.
- **Préférer les patterns par mot-clé** (`Point(x=0, y=y)`) pour les classes avec plus de 2-3 attributs : c'est plus robuste si l'ordre change.
- **Utiliser des dataclasses ou des Enum** pour clarifier l'intention et éviter le piège du capture.

### À ne pas faire

- **Comparer un nom simple à une constante** : `case MA_CONSTANTE` capture, ne compare pas. Utilisez `Module.MA_CONSTANTE` ou une Enum.
- **Mettre de la logique complexe dans les guards** : si un guard dépasse une ligne lisible, extrayez-le dans une fonction.
- **Abuser de l'imbrication** : au-delà de 2 niveaux, la lisibilité chute. Découpez la logique en plusieurs fonctions qui font chacune un `match`.
- **Utiliser `match` comme un dispatch de méthodes** : si vos `case` se limitent à appeler une méthode par type, le polymorphisme classique est probablement plus adapté.

---

## Quand l'utiliser, quand l'éviter

`match / case` brille quand :
- Vous traitez des structures de données (dict JSON, AST, message protocolaire).
- Vous avez besoin de comparer ET d'extraire dans le même geste.
- Vous travaillez sur une hiérarchie de classes (dataclasses, Enum).

Un simple `if / elif` reste préférable quand :
- Vous comparez 2 ou 3 valeurs sans extraction.
- Les conditions sont des expressions complexes qui ne tiennent pas dans un pattern.
- Vous voulez rester compatible avec Python 3.9 ou plus ancien.

---

## Conclusion

Le pattern matching est l'un des ajouts les plus puissants de Python 3.10. Il transforme du code défensif (`isinstance`, `.get()`, variables temporaires) en code déclaratif qui exprime la structure attendue des données.

**Points clés :**
- `match / case` combine comparaison et déstructuration en une seule expression.
- Les patterns existent pour les littéraux, les séquences, les dictionnaires et les classes.
- Un nom simple dans un `case` est toujours une capture, jamais une comparaison.
- Les dataclasses et les Enums se marient particulièrement bien avec `match / case`.
- Pour comparer à une constante, qualifiez-la (`Module.NOM`) ou utilisez une Enum.

---

## Pour aller plus loin

- [PEP 634 - Structural Pattern Matching: Specification](https://peps.python.org/pep-0634/)
- [PEP 635 - Structural Pattern Matching: Motivation and Rationale](https://peps.python.org/pep-0635/)
- [PEP 636 - Structural Pattern Matching: Tutorial](https://peps.python.org/pep-0636/)
- [Documentation officielle : match statement](https://docs.python.org/3/reference/compound_stmts.html#the-match-statement)

## Voir aussi

- [Pattern matching en Java moderne]({% post_url 2025-10-23-Pattern-matching-en-Java-moderne %})
- [Python : Comment utiliser les décorateurs]({% post_url 2026-05-14-Python-les-decorateurs %})
- [Python : f-strings, formatage de chaînes]({% post_url 2026-02-09-Python-f-strings-formatage-chaines %})
- [Python : Comment tester son code avec pytest]({% post_url 2026-04-06-Comment-tester-son-code-python-avec-pytest %})
- [Sealed classes en Java]({% post_url 2026-01-14-Sealed-classes-en-Java %})
