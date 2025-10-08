---
layout: article
title: "Python : Comment faire des group by"
author: Pierre Chopinet
tags:
  - python
  - data
  - itertools
  - pandas
  - tutoriel
---

Regrouper des données par clé (faire un « group by ») est une opération courante : compter des occurrences, agréger des montants par catégorie, calculer des moyennes par groupe, etc. En Python, il existe plusieurs façons de procéder selon la taille des données, le besoin d’agrégation et vos dépendances.
<!--more-->

Dans cet article :
- Groupement simple avec `defaultdict`
- Groupement en flux avec `itertools.groupby`
- Comptages avec `Counter`
- Multi‑clés et agrégations personnalisées
- « Group by » haut niveau avec `pandas`
- Pièges et bonnes pratiques

## Jeu de données d’exemple

Nous utiliserons une petite liste de ventes :

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

## 1) Groupement simple avec collections.defaultdict

La manière la plus directe pour regrouper des éléments par clé est d’utiliser `defaultdict(list)` :

```python
from collections import defaultdict

par_ville = defaultdict(list)
for v in ventes:
    par_ville[v["ville"]].append(v)

# Accès
print(par_ville["Paris"])  # -> liste des ventes de Paris
```

- Avantages : simple, lisible, ne nécessite pas de tri préalable.
- Inconvénients : stocke toutes les lignes en mémoire dans des listes.

Variation avec `setdefault` (si vous ne voulez pas importer `defaultdict`) :

```python
groupes = {}
for v in ventes:
    groupes.setdefault(v["ville"], []).append(v)
```

## 2) Groupement avec itertools.groupby

`itertools.groupby` groupe les éléments consécutifs ayant la même clé. Il exige que les données soient triées par la clé de groupement, sinon les groupes seront fragmentés.

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

- Avantages : fonctionne en flux (chaque groupe est produit à la volée), utile pour gros volumes si la source est déjà triée.
- Inconvénients : nécessite un tri (O(n log n)) ou une source déjà triée. Les éléments identiques, mais non contigus ne sont pas fusionnés.

Astuce : groupement par plusieurs champs en une fois en utilisant une clé composée :

```python
cles = ("ville", "produit")
ventes_triees = sorted(ventes, key=itemgetter(*cles))
for cle, grp in groupby(ventes_triees, key=itemgetter(*cles)):
    ville, produit = cle
    total_qte = sum(v["qte"] for v in grp)
    print((ville, produit), "->", total_qte)
```

## 3) Compter rapidement avec collections.Counter

Si vous voulez seulement compter le nombre d’occurrences d’une clé (et pas regrouper les lignes), `Counter` est très pratique :

```python
from collections import Counter

# Combien de ventes par ville ?
compte = Counter(v["ville"] for v in ventes)
print(compte)  # Counter({'Paris': 3, 'Lyon': 2, 'Nantes': 1})
```

Pour compter par clé multiple, utilisez un tuple comme clé :

```python
compte_ville_produit = Counter((v["ville"], v["produit"]) for v in ventes)
```

## 4) Agrégations personnalisées (sommes, moyennes…)

Pour agréger des mesures par groupe (ex. chiffre d’affaires par ville), on peut accumuler des totaux dans un dictionnaire :

```python
from collections import defaultdict

ca_par_ville = defaultdict(float)
for v in ventes:
    ca_par_ville[v["ville"]] += v["qte"] * v["prix"]

print(dict(ca_par_ville))
```

Moyenne par groupe (accumuler somme et compte) :

```python
from collections import defaultdict

somme_et_n = defaultdict(lambda: [0.0, 0])  # [somme, n]
for v in ventes:
    d = somme_et_n[v["ville"]]
    d[0] += v["qte"] * v["prix"]
    d[1] += 1

moy_par_ville = {ville: somme / n for ville, (somme, n) in somme_et_n.items()}
```

## 5) Group by haut niveau avec pandas

Lorsque vous manipulez des données en tableau, `pandas` offre un `groupby` très puissant et concis.

```python
import pandas as pd

df = pd.DataFrame(ventes)

# Somme des quantités par ville
print(df.groupby("ville")["qte"].sum())

# Agrégations multiples (créer d'abord une colonne chiffre d'affaires)
df = df.assign(ca=df["qte"] * df["prix"])
agg = df.groupby("ville").agg(
    total_qte=("qte", "sum"),
    total_ca=("ca", "sum"),
)
print(agg)
```

Agrégation sur plusieurs clés et plusieurs mesures :

```python
res = (
    df.groupby(["ville", "produit"]).agg(
        total_qte=("qte", "sum"),
        prix_moyen=("prix", "mean"),
    )
)
print(res)
```

Astuces pandas :
- `as_index=False` pour conserver les colonnes de groupement comme colonnes normales.
- `reset_index()` pour aplatir l’index après un groupby.
- `pivot_table` est une alternative pratique pour des tableaux croisés.

## 6) Choisir la bonne approche

- Petites/moyennes données en mémoire, besoin de groupes réels : `defaultdict(list)`.
- Données triées ou besoin de streaming par groupe : `itertools.groupby` (après tri si nécessaire).
- Simple comptage d’occurrences : `Counter`.
- Agrégations numériques personnalisées sans conserver les lignes : dictionnaires d’accumulateurs.
- Données tabulaires et besoins analytiques avancés : `pandas.DataFrame.groupby`.

## Pièges et bonnes pratiques

- `itertools.groupby` regroupe seulement les éléments consécutifs : triez par la même clé avant de grouper.
- Pour plusieurs clés : utilisez des tuples comme clés (`(ville, produit)`) ou `itemgetter(*cles)`.
- Si l’ordre d’apparition d’origine est important, préférez `defaultdict` + accumulation; `groupby` après tri perd l’ordre initial.
- Pour de très gros volumes non triés, envisagez une base embarquée (`sqlite3`), `pandas` en mode chunk, ou un tri externe.
- Évitez d’empiler de gros objets dans des listes si vous ne les réutilisez pas : cumulez directement les agrégats.
- Utilisez `operator.itemgetter`/`attrgetter` pour des clés rapides et lisibles.

## Conclusion

Python offre plusieurs stratégies de « group by », du plus bas niveau (`defaultdict`, `groupby`) jusqu’au haut niveau avec `pandas`.
Choisissez la méthode en fonction de votre volume de données, du besoin de conserver les lignes ou non, et des agrégations à réaliser.

## Voir aussi

- [Comment merger deux DataFrame pandas]({% post_url 2025-08-31-Comment-merger-deux-dataframe-pandas %})
- [Python : Comment sauvegarder et charger un dataframe Pandas avec Excel (ou du csv)]({% post_url 2023-12-28-Comment-sauvegarder-un-dataframe-pandas %})
- [Python : Comment sauvegarder des tableaux NumPy]({% post_url 2022-01-25-Comment-sauvegarder-un-tableau-numpy %})
