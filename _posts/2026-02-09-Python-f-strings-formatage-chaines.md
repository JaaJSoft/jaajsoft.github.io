---
layout: article
title: "Python : Comment utiliser les f-strings"
description: "Formater des chaînes en Python avec les f-strings : expressions, nombres, alignement, dates, accolades, débogage avec = et limites avant Python 3.12."
tags:
  - python
  - strings
author: Pierre Chopinet
---

Depuis Python 3.6, il suffit de préfixer une chaîne par `f` pour y insérer des variables entre accolades : `f"Bonjour {nom}"`. Nous allons faire le tour de ce qu'on peut mettre dans ces accolades, du simple nom de variable au formatage des nombres, des dates et des colonnes alignées.
<!--more-->

Dans cet article :
- Syntaxe de base
- Des expressions entre les accolades
- Formater les nombres
- Largeur et alignement
- Formater des dates
- Afficher des accolades
- Déboguer avec `=`
- Backslashes et guillemets avant Python 3.12
- Quand ne pas utiliser une f-string

Pré-requis : Python 3.6 ou plus récent (3.8 pour la syntaxe `=` de débogage). Les exemples ont été testés avec Python 3.13.

## Syntaxe de base

On ajoute un `f` devant les guillemets, et chaque expression entre accolades est remplacée par sa valeur. Voici le même message construit avec les trois méthodes de formatage de Python :

```python
nom = "Alice"
age = 30

# Ancienne méthode avec %
message = "Bonjour %s, vous avez %d ans." % (nom, age)

# Méthode .format()
message = "Bonjour {}, vous avez {} ans.".format(nom, age)

# f-string
message = f"Bonjour {nom}, vous avez {age} ans."
print(message)  # Bonjour Alice, vous avez 30 ans.
```

Les trois versions donnent la même chaîne. Avec `%`, il faut indiquer le type de chaque valeur (`%s`, `%d`...) et fournir exactement le bon nombre d'arguments, sinon on obtient une `TypeError`. Avec `format()`, les valeurs sont listées à la fin, loin de l'endroit où elles apparaissent. La f-string, elle, se lit dans l'ordre.

Elle est aussi plus rapide : mesurée avec `timeit` sous Python 3.13, elle prend environ deux fois moins de temps que `format()` et un tiers de moins que `%` sur cet exemple. Python la transforme dès la compilation en une suite d'opérations simples (on peut le voir avec le module `dis`), alors que `%` et `format()` analysent la chaîne de format à chaque exécution.

Le préfixe fonctionne aussi avec les triples guillemets, pour un message sur plusieurs lignes :

```python
nom = "Alice"
age = 30
ville = "Paris"

message = f"""
Bonjour {nom},
Vous avez {age} ans et vous habitez à {ville}.
Bienvenue sur notre plateforme !
"""

print(message)
```

## Des expressions entre les accolades

Entre les accolades, on n'est pas limité aux noms de variables : n'importe quelle expression Python est acceptée, que ce soit un calcul, une condition ou un appel de fonction.

```python
x = 10
y = 5

print(f"La somme de {x} et {y} est {x + y}")
# La somme de 10 et 5 est 15

print(f"Le produit est {x * y}")
# Le produit est 50

print(f"{x} est {'pair' if x % 2 == 0 else 'impair'}")
# 10 est pair
```

Remarquez les guillemets simples autour de `'pair'` : jusqu'à Python 3.11, on ne pouvait pas réutiliser à l'intérieur des accolades le guillemet qui délimite la f-string (on y revient plus bas).

Les appels de fonctions et de méthodes fonctionnent de la même façon :

```python
def prix_ttc(prix_ht, tva=0.20):
    return prix_ht * (1 + tva)

prix = 100
print(f"Prix TTC : {prix_ttc(prix):.2f}€")
# Prix TTC : 120.00€
```

```python
texte = "python"
print(f"Majuscule : {texte.upper()}")
# Majuscule : PYTHON

print(f"Longueur : {len(texte)} caractères")
# Longueur : 6 caractères
```

Le `:.2f` après l'appel de `prix_ttc` arrondit le résultat à deux décimales : ces formats sont détaillés dans la section suivante.

Ce n'est pas une raison pour tout écrire dans les accolades. Dès que l'expression devient longue, mieux vaut la calculer avant dans une variable :

```python
panier = [{"prix": 12.5, "qte": 2}, {"prix": 1.2, "qte": 5}]

# Difficile à lire
print(f"Total : {sum([p['prix'] * p['qte'] for p in panier]):.2f}€")

# Plus clair
total = sum(p['prix'] * p['qte'] for p in panier)
print(f"Total : {total:.2f}€")
```

## Formater les nombres

Après l'expression, un `:` introduit une spécification de format. C'est le même mini-langage que celui de `format()`.

### Décimales

`.2f` affiche le nombre avec deux décimales, en arrondissant :

```python
pi = 3.141592653589793

print(f"{pi:.2f}")  # 3.14 (2 décimales)
print(f"{pi:.4f}")  # 3.1416 (4 décimales)
print(f"{pi:.0f}")  # 3 (entier)
```

### Séparateurs de milliers

```python
nombre = 1234567.89

print(f"{nombre:,.2f}")     # 1,234,567.89 (virgule US)
print(f"{nombre:_.2f}")     # 1_234_567.89 (underscore)
```

Il n'existe que ces deux séparateurs de milliers. Attention au faux ami `{nombre: .2f}` : l'espace n'y est pas un séparateur de milliers mais une option de signe, qui met une espace devant les nombres positifs, là où les négatifs ont leur `-`.

Pour obtenir le format français, avec des espaces entre les milliers et une virgule avant les décimales, le plus simple est de remplacer les caractères après coup :

```python
print(f"{nombre:,.2f}".replace(",", " ").replace(".", ","))  # 1 234 567,89
```

L'autre solution est le format `n`, qui utilise les séparateurs de la locale courante. Il faut d'abord la choisir avec `locale.setlocale`, car Python ne reprend pas celle du système par défaut :

```python
import locale
locale.setlocale(locale.LC_ALL, 'fr_FR.UTF-8')

nombre_entier = 1234567
print(f"{nombre_entier:n}")  # 1 234 567

print(f"{nombre:n}")    # 1,23457e+06
print(f"{nombre:.9n}")  # 1 234 567,89
```

Sur un float, `n` se comporte comme le format `g` : six chiffres significatifs par défaut, d'où la notation scientifique pour `1234567.89`. Il faut donc préciser le nombre de chiffres significatifs voulus, ici 9.

Avec la locale française de la glibc (testé avec la version 2.39, sous Ubuntu), le séparateur de milliers est une espace fine insécable (U+202F) et non une espace classique : à garder en tête si vous comparez ou découpez la chaîne obtenue. Si la locale n'est pas installée sur la machine, `setlocale` lève une exception `locale.Error: unsupported locale setting`. Sous Windows, la locale française peut s'appeler `'French_France.1252'` au lieu de `'fr_FR.UTF-8'`.

### Pourcentages, notation scientifique et bases

Le format `%` multiplie la valeur par 100 et ajoute le signe pour cent :

```python
taux = 0.1547

print(f"Taux : {taux:.2%}")  # Taux : 15.47%
print(f"Taux : {taux:.1%}")  # Taux : 15.5%
```

`e` passe en notation scientifique :

```python
grand_nombre = 1234567890

print(f"{grand_nombre:e}")   # 1.234568e+09
print(f"{grand_nombre:.2e}") # 1.23e+09
```

Enfin, `b`, `o`, `x` et `X` affichent un entier en binaire, en octal ou en hexadécimal. Avec `#`, on ajoute le préfixe correspondant (`0b`, `0o` ou `0x`) :

```python
nombre = 42

print(f"Binaire : {nombre:b}")    # Binaire : 101010
print(f"Octal : {nombre:o}")      # Octal : 52
print(f"Hexadécimal : {nombre:x}")  # Hexadécimal : 2a
print(f"Hexadécimal (MAJ) : {nombre:X}")  # Hexadécimal (MAJ) : 2A
print(f"Hex avec préfixe : {nombre:#x}")   # Hex avec préfixe : 0x2a
```

## Largeur et alignement

Un nombre seul après les deux-points fixe la largeur minimale du champ. Par défaut, le texte est aligné à gauche et les nombres à droite :

```python
nom = "Alice"
age = 30

print(f"{nom:10} | {age:3}")
# Alice      |  30
```

Pour choisir l'alignement, on utilise `<` (à gauche), `>` (à droite) ou `^` (centré) :

```python
texte = "Python"

print(f"{texte:<10}")  # Python     (gauche)
print(f"{texte:>10}")  #     Python (droite)
print(f"{texte:^10}")  #   Python   (centré)
```

Le caractère placé juste avant le signe d'alignement sert à remplir l'espace libre :

```python
nombre = 42

print(f"{nombre:0>5}")  # 00042 (zéros à gauche)
print(f"{nombre:*<5}")  # 42*** (étoiles à droite)
print(f"{nombre:-^7}")  # --42--- (centré, avec des tirets)
```

La largeur se combine avec la précision :

```python
prix = 12.5

print(f"{prix:10.2f}")   #      12.50 (10 caractères, 2 décimales)
print(f"{prix:0>10.2f}") # 0000012.50 (complété avec des zéros)
```

Avec tout ça, on peut afficher un petit tableau aligné dans le terminal :

```python
produits = [
    ("Livre", 12.50, 2),
    ("Stylo", 1.20, 5),
    ("Cahier", 3.00, 4),
]

print(f"{'Produit':<15} {'Prix':>8} {'Qté':>5} {'Total':>8}")
print("-" * 40)
for nom, prix, qte in produits:
    total = prix * qte
    print(f"{nom:<15} {prix:>8.2f} {qte:>5} {total:>8.2f}")
```

Ce qui donne :

```
Produit             Prix   Qté    Total
----------------------------------------
Livre              12.50     2    25.00
Stylo               1.20     5     6.00
Cahier              3.00     4    12.00
```

La largeur et la précision peuvent aussi venir de variables : la spécification de format accepte elle-même des champs entre accolades, qui sont remplacés avant le formatage.

```python
nombre = 1234.5678
precision = 2

print(f"{nombre:.{precision}f}")
# 1234.57

texte = "Python"
largeur = 15
alignement = "^"  # centré

print(f"{texte:{alignement}{largeur}}")
#     Python
```

## Formater des dates

Les objets `datetime` acceptent directement les codes de `strftime` après les deux-points :

```python
from datetime import datetime

moment = datetime(2026, 2, 9, 14, 30, 45)

print(f"Date : {moment:%Y-%m-%d}")
# Date : 2026-02-09

print(f"Heure : {moment:%H:%M:%S}")
# Heure : 14:30:45

print(f"Date complète : {moment:%A %d %B %Y}")
# Date complète : Monday 09 February 2026

print(f"Format court : {moment:%d/%m/%Y %H:%M}")
# Format court : 09/02/2026 14:30
```

Le principe est le même avec `datetime.now()`, ce qui est pratique pour horodater un message : `print(f"[{datetime.now():%Y-%m-%d %H:%M:%S}] {message}")`.

Les noms des jours et des mois dépendent de la locale, et tant qu'on n'a pas appelé `setlocale`, Python les affiche en anglais. Pour les avoir en français :

```python
import locale
locale.setlocale(locale.LC_TIME, 'fr_FR.UTF-8')

print(f"{moment:%A %d %B %Y}")
# lundi 09 février 2026
```

Les remarques faites pour le format `n` s'appliquent ici aussi : la locale doit être installée, et sous Windows elle peut s'appeler `'French_France.1252'`.

## Afficher des accolades

{% raw %}
Pour écrire une accolade dans une f-string sans qu'elle soit interprétée, on la double : `{{` affiche `{` et `}}` affiche `}`.

```python
x = 10

print(f"La variable x vaut {x}")
# La variable x vaut 10

print(f"Pour afficher {{x}}, écrivez {{{{x}}}}")
# Pour afficher {x}, écrivez {{x}}

print(f"{{{x}}}")
# {10}
```

Dans le dernier exemple, `{{` donne l'accolade ouvrante, `{x}` la valeur de `x` et `}}` l'accolade fermante.
{% endraw %}

## Déboguer avec `=`

Depuis Python 3.8, un `=` placé après l'expression affiche l'expression elle-même, puis sa valeur :

```python
x = 10
y = 20

print(f"{x=}, {y=}, {x + y=}")
# x=10, y=20, x + y=30
```

C'est pratique pour un `print` de débogage rapide, sans recopier le nom de chaque variable :

```python
def calculer_total(prix, quantite):
    total = prix * quantite
    print(f"{prix=}, {quantite=}, {total=}")
    return total

calculer_total(12.5, 3)
# prix=12.5, quantite=3, total=37.5
```

La valeur est affichée avec `repr()`, si bien qu'une chaîne apparaît entre guillemets : `f"{nom=}"` donne `nom='Alice'`. Les espaces autour du `=` sont conservés tels quels : `f"{x = }"` affiche `x = 10`.

## Backslashes et guillemets avant Python 3.12

Jusqu'à Python 3.11, la partie entre accolades avait deux limites : elle ne pouvait pas contenir de backslash, ni le guillemet qui délimite la chaîne.

```python
items = ["pomme", "poire", "cerise"]

# Erreur avant Python 3.12
print(f"Liste : {'\n'.join(items)}")
# SyntaxError: f-string expression part cannot include a backslash

# Solution compatible avec toutes les versions : passer par une variable
saut = "\n"
print(f"Liste : {saut.join(items)}")
```

```python
# Erreur de syntaxe avant Python 3.12
print(f"Message : {"Hello"}")
# SyntaxError: f-string: expecting '}'

# Solution : alterner les guillemets
print(f"Message : {'Hello'}")
print(f'Message : {"Hello"}')
```

La PEP 701, arrivée avec Python 3.12, a supprimé ces deux restrictions : les lignes marquées en erreur fonctionnent avec une version récente. Les contournements restent utiles si votre code doit aussi tourner sur Python 3.11 ou une version plus ancienne.

## Quand ne pas utiliser une f-string

Une f-string est évaluée immédiatement, à l'endroit où elle est écrite. Si le modèle de texte existe avant les valeurs, parce qu'il est lu dans un fichier de configuration par exemple, il faut revenir à `format()` avec des champs nommés :

```python
modele = "Bonjour {nom}, vous avez {age} ans."

print(modele.format(nom="Alice", age=30))
print(modele.format(nom="Bob", age=25))
```

```
Bonjour Alice, vous avez 30 ans.
Bonjour Bob, vous avez 25 ans.
```

Et quelle que soit la méthode, on ne construit pas une requête SQL en y insérant des valeurs : il faut passer par les requêtes paramétrées du pilote de base de données (les `?` de `sqlite3` par exemple). Sinon, une simple apostrophe dans une valeur casse la requête, et un utilisateur malveillant peut y injecter son propre SQL.

## Voir aussi

- [Python : Comment utiliser les décorateurs]({% post_url 2026-05-14-Python-les-decorateurs %})
- [Python : Le pattern matching avec match et case]({% post_url 2026-05-18-Python-pattern-matching-avec-match-et-case %})
- [Documentation officielle des f-strings](https://docs.python.org/3/reference/lexical_analysis.html#f-strings)
- [Le mini-langage de spécification de format](https://docs.python.org/3/library/string.html#format-specification-mini-language)
- [PEP 498 - Literal String Interpolation](https://peps.python.org/pep-0498/)
- [PEP 701 - Syntactic formalization of f-strings](https://peps.python.org/pep-0701/)
