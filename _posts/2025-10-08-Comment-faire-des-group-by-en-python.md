---
layout: article
title: "Python : Comment faire des group by"
description: "Faire des group by en Python : defaultdict, itertools.groupby, Counter, sommes et moyennes par groupe, groupby et pivot_table de pandas, et gros volumes."
tags:
  - python
  - data
  - itertools
  - pandas
author: Pierre Chopinet
---

Regrouper des données par clé, le "group by" de SQL, revient tout le temps : compter des occurrences, additionner des montants par catégorie, calculer une moyenne par groupe. Nous allons voir les différentes façons de le faire en Python, de la bibliothèque standard jusqu'à pandas, sur un même jeu de données.
<!--more-->

Dans cet article :
- Le jeu de données
- Regrouper avec `defaultdict`
- Regrouper des données triées avec `itertools.groupby`
- Compter avec `Counter`
- Calculer des sommes et des moyennes
- Le `groupby` de pandas
- Quand les données ne tiennent pas en mémoire

Pré-requis : connaître les listes et les dictionnaires Python. Les exemples ont été testés avec Python 3.13 et pandas 3.0.5.

## Le jeu de données

Tous les exemples utilisent la même petite liste de ventes :

```python
ventes = [
    {"ville": "Paris",   "produit": "Livre",   "qte": 2, "prix": 12.5},
    {"ville": "Lyon",    "produit": "Stylo",   "qte": 5, "prix": 1.2},
    {"ville": "Paris",   "produit": "Stylo",   "qte": 3, "prix": 1.2},
    {"ville": "Nantes",  "produit": "Livre",   "qte": 1, "prix": 12.5},
    {"ville": "Paris",   "produit": "Cahier",  "qte": 4, "prix": 3.0},
    {"ville": "Lyon",    "produit": "Livre",   "qte": 2, "prix": 12.5},
]
```

## Regrouper avec `defaultdict`

Le plus direct pour regrouper des éléments par clé est un `defaultdict(list)` : quand on accède à une clé qui n'existe pas encore, il l'initialise avec une liste vide.

```python
from collections import defaultdict

par_ville = defaultdict(list)
for v in ventes:
    par_ville[v["ville"]].append(v)

# Accès
print(par_ville["Paris"])  # -> liste des ventes de Paris
```

On récupère les trois ventes de Paris :

```
[{'ville': 'Paris', 'produit': 'Livre', 'qte': 2, 'prix': 12.5}, {'ville': 'Paris', 'produit': 'Stylo', 'qte': 3, 'prix': 1.2}, {'ville': 'Paris', 'produit': 'Cahier', 'qte': 4, 'prix': 3.0}]
```

Pas besoin de trier les données avant, et les villes restent dans l'ordre de leur première apparition, puisqu'un dictionnaire conserve l'ordre d'insertion. Par contre, toutes les lignes sont gardées en mémoire dans les listes.

On peut se passer de l'import avec `setdefault`, qui renvoie la valeur associée à la clé après l'avoir initialisée si elle n'existait pas :

```python
groupes = {}
for v in ventes:
    groupes.setdefault(v["ville"], []).append(v)
```

## Regrouper des données triées avec `itertools.groupby`

`itertools.groupby` fonctionne comme la commande `uniq` d'Unix : il regroupe les éléments consécutifs qui ont la même clé. Il faut donc trier les données sur cette clé avant de les lui passer :

```python
from itertools import groupby
from operator import itemgetter

# Trier d'abord par la clé
ventes_triees = sorted(ventes, key=itemgetter("ville"))

# Grouper
for ville, groupe_iter in groupby(ventes_triees, key=itemgetter("ville")):
    groupe = list(groupe_iter)  # matérialiser si besoin de réutiliser
    print(ville, "->", len(groupe), "lignes")
```

Ce qui donne :

```
Lyon -> 2 lignes
Nantes -> 1 lignes
Paris -> 3 lignes
```

`itemgetter("ville")` fait la même chose que `lambda v: v["ville"]`, en plus court. Pour grouper des objets sur un attribut, `operator.attrgetter` joue le même rôle.

Sans le tri, `groupby` crée un nouveau groupe à chaque fois que la ville change :

```python
for ville, groupe_iter in groupby(ventes, key=itemgetter("ville")):
    print(ville, "->", len(list(groupe_iter)), "lignes")
```

```
Paris -> 1 lignes
Lyon -> 1 lignes
Paris -> 1 lignes
Nantes -> 1 lignes
Paris -> 1 lignes
Lyon -> 1 lignes
```

Attention aussi au `list(groupe_iter)` du premier exemple : chaque groupe est un itérateur qui partage la source avec `groupby`. Dès qu'on passe au groupe suivant, le précédent n'est plus accessible, il faut donc le stocker dans une liste si on veut s'en resservir.

L'intérêt de `groupby` est de travailler en flux : les groupes sont produits au fur et à mesure, sans tout charger en mémoire. C'est utile quand la source est déjà triée, par exemple un gros fichier qu'on lit ligne par ligne. Dans le cas contraire, il faut payer le tri (en O(n log n)), et les groupes sortent dans l'ordre des clés et non plus dans l'ordre d'apparition.

Pour grouper sur plusieurs champs, on passe plusieurs noms à `itemgetter`, qui renvoie alors un tuple :

```python
cles = ("ville", "produit")
ventes_triees = sorted(ventes, key=itemgetter(*cles))
for cle, grp in groupby(ventes_triees, key=itemgetter(*cles)):
    ville, produit = cle
    total_qte = sum(v["qte"] for v in grp)
    print((ville, produit), "->", total_qte)
```

```
('Lyon', 'Livre') -> 2
('Lyon', 'Stylo') -> 5
('Nantes', 'Livre') -> 1
('Paris', 'Cahier') -> 4
('Paris', 'Livre') -> 2
('Paris', 'Stylo') -> 3
```

## Compter avec `Counter`

Si on veut seulement savoir combien de fois chaque clé apparaît, sans garder les lignes, `Counter` s'en charge :

```python
from collections import Counter

# Combien de ventes par ville ?
compte = Counter(v["ville"] for v in ventes)
print(compte)  # Counter({'Paris': 3, 'Lyon': 2, 'Nantes': 1})
```

`compte.most_common(1)` renvoie la ville qui a le plus de ventes, ici `[('Paris', 3)]`.

Pour compter sur plusieurs champs, on utilise là aussi un tuple comme clé :

```python
compte_ville_produit = Counter((v["ville"], v["produit"]) for v in ventes)
```

## Calculer des sommes et des moyennes

Pour agréger une valeur par groupe, le chiffre d'affaires par ville par exemple, inutile de stocker les lignes : on cumule directement les totaux dans un dictionnaire.

```python
from collections import defaultdict

ca_par_ville = defaultdict(float)
for v in ventes:
    ca_par_ville[v["ville"]] += v["qte"] * v["prix"]

print(dict(ca_par_ville))
```

```
{'Paris': 40.6, 'Lyon': 31.0, 'Nantes': 12.5}
```

Pour une moyenne, il faut garder deux choses par groupe : la somme et le nombre d'éléments. Ici, on calcule le montant moyen d'une vente dans chaque ville :

```python
from collections import defaultdict

somme_et_n = defaultdict(lambda: [0.0, 0])  # [somme, n]
for v in ventes:
    d = somme_et_n[v["ville"]]
    d[0] += v["qte"] * v["prix"]
    d[1] += 1

moy_par_ville = {ville: somme / n for ville, (somme, n) in somme_et_n.items()}
print(moy_par_ville)
```

```
{'Paris': 13.533333333333333, 'Lyon': 15.5, 'Nantes': 12.5}
```

## Le `groupby` de pandas

Dès que les données sont sous forme de tableau, pandas est plus pratique : son `groupby` regroupe et agrège en une seule expression.

```python
import pandas as pd

df = pd.DataFrame(ventes)

# Somme des quantités par ville
print(df.groupby("ville")["qte"].sum())
```

```
ville
Lyon      7
Nantes    1
Paris     9
Name: qte, dtype: int64
```

Avec `agg`, on calcule plusieurs agrégations d'un coup en donnant un nom à chaque colonne du résultat :

```python
# Agrégations multiples (créer d'abord une colonne chiffre d'affaires)
df = df.assign(ca=df["qte"] * df["prix"])
agg = df.groupby("ville").agg(
    total_qte=("qte", "sum"),
    total_ca=("ca", "sum"),
)
print(agg)
```

```
        total_qte  total_ca
ville                      
Lyon            7      31.0
Nantes          1      12.5
Paris           9      40.6
```

Pour grouper sur plusieurs colonnes, on passe une liste :

```python
res = (
    df.groupby(["ville", "produit"]).agg(
        total_qte=("qte", "sum"),
        prix_moyen=("prix", "mean"),
    )
)
print(res)
```

```
                total_qte  prix_moyen
ville  produit                       
Lyon   Livre            2        12.5
       Stylo            5         1.2
Nantes Livre            1        12.5
Paris  Cahier           4         3.0
       Livre            2        12.5
       Stylo            3         1.2
```

Par défaut, pandas trie les groupes par clé (`sort=False` pour garder l'ordre d'apparition) et les colonnes de groupement deviennent l'index du résultat. Pour les récupérer comme des colonnes normales, on passe `as_index=False` à `groupby`, ou on appelle `reset_index()` sur le résultat :

```python
print(res.reset_index())
```

```
    ville produit  total_qte  prix_moyen
0    Lyon   Livre          2        12.5
1    Lyon   Stylo          5         1.2
2  Nantes   Livre          1        12.5
3   Paris  Cahier          4         3.0
4   Paris   Livre          2        12.5
5   Paris   Stylo          3         1.2
```

Enfin, pour un tableau croisé avec les villes en lignes et les produits en colonnes, `pivot_table` donne un résultat plus lisible :

```python
print(df.pivot_table(index="ville", columns="produit", values="qte", aggfunc="sum", fill_value=0))
```

```
produit  Cahier  Livre  Stylo
ville                        
Lyon          0      2      5
Nantes        0      1      0
Paris         4      2      3
```

## Quand les données ne tiennent pas en mémoire

`defaultdict(list)` et pandas chargent toutes les lignes en mémoire. `Counter` et les dictionnaires d'accumulateurs, eux, peuvent consommer les lignes une par une, depuis un fichier lu ligne par ligne par exemple : ils ne gardent qu'une entrée par clé, et tiennent donc tant que le nombre de clés distinctes reste raisonnable. `itertools.groupby` va encore plus loin sur une source déjà triée, puisqu'il n'a besoin que du groupe en cours.

Au-delà, il faut changer d'approche : charger les données dans une base SQLite (module `sqlite3`) et faire un vrai `GROUP BY`, lire le fichier par morceaux avec pandas (`read_csv` avec le paramètre `chunksize`) en cumulant les résultats de chaque morceau, ou trier le fichier sur disque avant de le parcourir avec `itertools.groupby`.

## Voir aussi

- [Java : Comment faire des group by]({% post_url 2026-01-11-Comment-faire-des-group-by-en-Java %})
- [Python : Comment merger deux DataFrame pandas]({% post_url 2025-08-31-Comment-merger-deux-dataframe-pandas %})
- [Automatiser le nettoyage de données avec pandas]({% post_url 2025-12-14-Automatiser-le-nettoyage-de-donnees-avec-pandas %})
- [Python : Comment sauvegarder et charger un dataframe Pandas avec Excel (ou du csv)]({% post_url 2023-12-28-Comment-sauvegarder-un-dataframe-pandas %})
- [Documentation du module collections (defaultdict, Counter)](https://docs.python.org/3/library/collections.html)
- [Documentation de itertools.groupby](https://docs.python.org/3/library/itertools.html#itertools.groupby)
- [Documentation de pandas.DataFrame.groupby](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.groupby.html)
