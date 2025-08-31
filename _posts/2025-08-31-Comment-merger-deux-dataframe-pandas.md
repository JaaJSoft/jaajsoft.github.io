---
layout: article
title: "Comment merger deux DataFrame pandas"
author: Pierre Chopinet
tags:
  - python
  - pandas
  - data
  - merge
  - tutoriel
---

Dans ce tutoriel, nous verrons pas à pas comment fusionner (merger) deux DataFrame avec pandas : syntaxe de base, types de jointures (inner, left, right, outer, cross), clés multiples, jointures sur l’index, gestion des doublons et colonnes homonymes, options utiles (`suffixes`, `validate`, `indicator`) et quelques conseils de performance.
<!--more-->

## Installation rapide

```bash
pip install pandas
```

Sous Windows (PowerShell) :

```powershell
python -m pip install pandas
```

---

## Jeu de données d’exemple

```python
import pandas as pd

# Table des clients
clients = pd.DataFrame({
    "client_id": [1, 2, 3, 4],
    "nom": ["Alice", "Bob", "Chloé", "David"],
    "ville": ["Lyon", "Paris", "Lille", "Lyon"],
})

# Table des commandes
commandes = pd.DataFrame({
    "id_client": [1, 1, 2, 5],   # Note: 5 n'existe pas côté clients
    "commande_id": [101, 102, 103, 104],
    "montant": [50.0, 20.0, 99.9, 15.5],
})

print(clients)
print(commandes)
```

Sortie :

```
   client_id    nom  ville
0          1  Alice   Lyon
1          2    Bob  Paris
2          3  Chloé  Lille
3          4  David   Lyon

   id_client  commande_id  montant
0          1          101     50.0
1          1          102     20.0
2          2          103     99.9
3          5          104     15.5
```

---

## 1) merge de base (inner join par défaut)

`pd.merge(left, right, ...)` ou `left.merge(right, ...)` réalise une jointure relationnelle.
Par défaut, c’est une jointure interne (inner) sur les colonnes communes.

Ici, nos clés n’ont pas le même nom (`client_id` vs `id_client`), on utilise `left_on`/`right_on` :

```python
# Inner join (par défaut) sur client_id == id_client
inner = pd.merge(
    clients, commandes,
    how="inner",
    left_on="client_id",
    right_on="id_client",
)
print(inner)
```

Sortie :

```
   client_id    nom  ville  id_client  commande_id  montant
0          1  Alice   Lyon          1          101     50.0
1          1  Alice   Lyon          1          102     20.0
2          2    Bob  Paris          2          103     99.9
```

Résultat : seules les lignes avec correspondance des deux côtés (clients 1 et 2).

---

## 2) Types de jointures

- inner (défaut) : intersection des clés.
- left : toutes les lignes de gauche, complétées si correspondance (sinon NaN).
- right : toutes les lignes de droite.
- outer : union des clés (left ∪ right).
- cross : produit cartésien (attention à la taille !).

Exemples :

```python
left = pd.merge(clients, commandes, how="left", left_on="client_id", right_on="id_client")
right = pd.merge(clients, commandes, how="right", left_on="client_id", right_on="id_client")
outer = pd.merge(clients, commandes, how="outer", left_on="client_id", right_on="id_client")
```

Sortie :

LEFT
```
   client_id    nom  ville  id_client  commande_id  montant
0          1  Alice   Lyon        1.0        101.0     50.0
1          1  Alice   Lyon        1.0        102.0     20.0
2          2    Bob  Paris        2.0        103.0     99.9
3          3  Chloé  Lille        NaN          NaN      NaN
4          4  David   Lyon        NaN          NaN      NaN
```

RIGHT
```
   client_id    nom  ville  id_client  commande_id  montant
0        1.0  Alice   Lyon          1          101     50.0
1        1.0  Alice   Lyon          1          102     20.0
2        2.0    Bob  Paris          2          103     99.9
3        NaN    NaN    NaN          5          104     15.5
```

OUTER
```
   client_id    nom  ville  id_client  commande_id  montant
0        1.0  Alice   Lyon        1.0        101.0     50.0
1        1.0  Alice   Lyon        1.0        102.0     20.0
2        2.0    Bob  Paris        2.0        103.0     99.9
3        3.0  Chloé  Lille        NaN          NaN      NaN
4        4.0  David   Lyon        NaN          NaN      NaN
5        NaN    NaN    NaN        5.0        104.0     15.5
```

Produit cartésien :

```python
# Sans clé: toutes les combinaisons (seulement si ça a du sens)
cross = pd.merge(clients, commandes, how="cross")
```

Sortie :

```
    client_id    nom  ville  id_client  commande_id  montant
0           1  Alice   Lyon          1          101     50.0
1           1  Alice   Lyon          1          102     20.0
2           1  Alice   Lyon          2          103     99.9
3           1  Alice   Lyon          5          104     15.5
4           2    Bob  Paris          1          101     50.0
5           2    Bob  Paris          1          102     20.0
6           2    Bob  Paris          2          103     99.9
7           2    Bob  Paris          5          104     15.5
8           3  Chloé  Lille          1          101     50.0
9           3  Chloé  Lille          1          102     20.0
10          3  Chloé  Lille          2          103     99.9
11          3  Chloé  Lille          5          104     15.5
12          4  David   Lyon          1          101     50.0
13          4  David   Lyon          1          102     20.0
14          4  David   Lyon          2          103     99.9
15          4  David   Lyon          5          104     15.5
```

---

## 3) Plusieurs clés (composite keys)

Vous pouvez joindre sur plusieurs colonnes en passant une liste :

```python
# Exemple artificiel avec une 2e clé
clients2 = clients.assign(pays="FR")
commandes2 = commandes.assign(pays=["FR", "FR", "FR", "FR"])  # même pays pour l'exemple

multi = pd.merge(
    clients2, commandes2,
    how="inner",
    left_on=["client_id", "pays"],
    right_on=["id_client", "pays"],
)
```

Sortie :

```
   client_id    nom  ville pays_x  id_client  commande_id  montant pays_y
0          1  Alice   Lyon     FR          1          101     50.0     FR
1          1  Alice   Lyon     FR          1          102     20.0     FR
2          2    Bob  Paris     FR          2          103     99.9     FR
```

---

## 4) Colonnes homonymes et suffixes

Quand des colonnes (autres que les clés) portent le même nom des deux côtés, pandas ajoute par défaut `_x` et `_y`. Vous pouvez contrôler cela avec `suffixes` :

```python
res = pd.merge(
    clients, commandes,
    how="inner",
    left_on="client_id", right_on="id_client",
    suffixes=("_client", "_cmd"),
)
```

---

## 5) indicator=True pour auditer la jointure

Ajoute une colonne `_merge` indiquant la provenance de chaque ligne : `left_only`, `right_only`, `both`.

```python
outer_audit = pd.merge(
    clients, commandes,
    how="outer",
    left_on="client_id", right_on="id_client",
    indicator=True,
)
print(outer_audit["_merge"].value_counts())
```

Utile pour vérifier ce qui matche/ne matche pas.

---

## 6) validate pour sécuriser le type de relation

`validate` permet de détecter des cardinalités inattendues :

- `"one_to_one"`
- `"one_to_many"` (ou `"1:m"`)
- `"many_to_one"` (ou `"m:1"`)
- `"many_to_many"` (ou `"m:m"`)

```python
# On s'attend à one-to-many (un client -> plusieurs commandes)
res = pd.merge(
    clients, commandes,
    left_on="client_id", right_on="id_client",
    how="left",
    validate="one_to_many",
)
```

Si la contrainte n’est pas respectée, pandas lève une `MergeError`.

---

## 7) Joindre via l’index (left_index/right_index) et DataFrame.join

Vous pouvez joindre sur l’index plutôt que sur des colonnes :

```python
clients_idx = clients.set_index("client_id")
commandes_idx = commandes.set_index("id_client")

# merge sur index
res_idx = pd.merge(
    clients_idx, commandes_idx,
    left_index=True, right_index=True,
    how="inner",
)

# équivalent pratique côté DataFrame
res_join = clients_idx.join(commandes_idx, how="inner")
```

`DataFrame.join` est un alias pratique pour joindre sur l’index (ou en mixant index/colonne avec `on=`).

---

## 8) Doublons de clés et lignes dupliquées

Les jointures many-to-many produisent un produit des correspondances. Parfois, c’est voulu; sinon, il faut dédoublonner ou agréger avant la jointure.

```python
# Supposons des doublons côté commandes sur (id_client)
cmd_dedup = commandes.drop_duplicates(subset=["id_client", "commande_id"])  # exemple

# Ou agréger avant la jointure (ex: montant total par client)
montants = commandes.groupby("id_client", as_index=False)["montant"].sum()
clients_total = clients.merge(montants, left_on="client_id", right_on="id_client", how="left")
```

---

## 9) Valeurs manquantes et lignes orphelines

- `how="left"` conserve tous les clients : les non-appariés auront des `NaN` côté commandes.
- `how="right"` conserve toutes les commandes : on peut voir les `id_client` sans fiche client.
- `how="outer"` montre tout et met `NaN` où il manque des infos.

On peut filtrer les orphelins avec `indicator=True` ou des tests sur `NaN` :

```python
outer_audit = pd.merge(
    clients, commandes,
    how="outer",
    left_on="client_id", right_on="id_client",
    indicator=True,
)
orphelins_cmd = outer_audit[outer_audit["_merge"] == "right_only"]  # commandes sans client
```

---

## 10) merge vs join vs concat

- `merge` : jointures relationnelles sur colonnes et/ou index.
- `join` : pratique pour joindre par l’index (syntaxe plus concise). Internellement, appelle `merge`.
- `concat` : empilement (vertical) ou juxtapositions (horizontal) sans logiques de clés.

Exemples :

```python
# concat vertical (ajouter des lignes)
all_clients = pd.concat([clients, pd.DataFrame({"client_id": [6], "nom": ["Emma"], "ville": ["Nice"]})], ignore_index=True)

# concat horizontal (alignement par index)
left = clients.set_index("client_id").sort_index()
right = commandes.set_index("id_client").sort_index()
horizontal = pd.concat([left, right], axis=1)
```

---

## 11) Performance et bonnes pratiques

- Assurez la compatibilité des types de vos clés avant la jointure (ex: `int64` vs `object`).
- Si peu de modalités répétées, convertissez les clés en `category` pour réduire mémoire et accélérer.
- Indexez/tri ez par les clés avant les grosses jointures : `set_index`, `sort_values`.
- Filtrez les colonnes nécessaires avec `[[...]]` avant de merger pour réduire le volume.
- Utilisez `validate` pour détecter tôt les cardinalités inattendues.
- Pour des jointures temporelles sur valeurs triées, regardez `pd.merge_asof` (approx. nearest key).
- Évitez `how="cross"` sur de grands DataFrame (croissance N×M).

---

## 12) Exemple complet reproductible

```python
import pandas as pd

clients = pd.DataFrame({
    "client_id": [1, 2, 3, 4],
    "nom": ["Alice", "Bob", "Chloé", "David"],
    "ville": ["Lyon", "Paris", "Lille", "Lyon"],
})
commandes = pd.DataFrame({
    "id_client": [1, 1, 2, 5],
    "commande_id": [101, 102, 103, 104],
    "montant": [50.0, 20.0, 99.9, 15.5],
})

# 1) inner
inner = clients.merge(commandes, how="inner", left_on="client_id", right_on="id_client")
print("INNER:\n", inner, sep="")

# 2) left + audit
left = pd.merge(clients, commandes, how="left", left_on="client_id", right_on="id_client", indicator=True)
print("\nLEFT + indicator:\n", left[["client_id", "commande_id", "_merge"]], sep="")

# 3) agrégation avant merge (total par client)
montants = commandes.groupby("id_client", as_index=False)["montant"].sum()
clients_total = clients.merge(montants, left_on="client_id", right_on="id_client", how="left").drop(columns=["id_client"])
print("\nTOTAL PAR CLIENT:\n", clients_total, sep="")

# 4) join sur index
ci = clients.set_index("client_id")
co = commandes.set_index("id_client")
join_idx = ci.join(co, how="left")
print("\nJOIN INDEX:\n", join_idx, sep="")
```

---

## FAQ

- Erreur `You are trying to merge on object and int64 columns` : convertissez les types (`astype`).
- Résultat plus grand que prévu : vous avez une jointure many-to-many. Ajoutez `validate="one_to_many"` ou dédoublonnez/agrégez.
- Colonnes en double après merge : utilisez `suffixes=("_g", "_d")` et/ou `drop` celles qui ne servent pas.

---

## Pour aller plus loin

- Documentation : (https://pandas.pydata.org/docs/reference/api/pandas.merge.html)
- `merge_asof` (temps triés) : (https://pandas.pydata.org/docs/reference/api/pandas.merge_asof.html)
- [Comment faire des requêtes HTTP en python avec requests]({% post_url 2020-05-22-Comment-faire-des-requetes-http-en-python-avec-requests %})
- [Comment sauvegarder un tableau numpy]({% post_url 2022-01-25-Comment-sauvegarder-un-tableau-numpy %})
- [Comment sauvegarder un dataframe pandas]({% post_url 2023-12-28-Comment-sauvegarder-un-dataframe-pandas %})


