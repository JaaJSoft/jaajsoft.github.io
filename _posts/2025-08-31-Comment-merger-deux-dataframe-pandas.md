---
layout: article
title: "Python : Comment merger deux DataFrame pandas"
description: "Fusionner deux DataFrame pandas avec merge : types de jointures, clés multiples, suffixes, indicator, validate, doublons et différences avec join et concat."
tags:
  - python
  - pandas
  - data
  - merge
author: Pierre Chopinet
---

Quand les données sont réparties dans plusieurs tables, des clients d'un côté et leurs commandes de l'autre par exemple, il faut les rassembler avant de pouvoir les analyser. Avec pandas, c'est le rôle de `merge`, l'équivalent des jointures SQL. Nous allons voir comment l'utiliser, et surtout comment vérifier que le résultat est bien celui qu'on attend.
<!--more-->

Dans cet article :
- Les données d'exemple
- Un premier merge
- Les types de jointures
- Joindre sur plusieurs colonnes
- Les colonnes en double et `suffixes`
- Retrouver les lignes sans correspondance
- Vérifier la relation avec `validate`
- Joindre sur l'index avec `join`
- Les clés en double
- Les clés doivent avoir le même type
- Empiler des tables avec `concat`

Pré-requis : Python 3 et pandas (`pip install pandas`). Les exemples ont été testés avec Python 3.13 et pandas 3.0.5.

## Les données d'exemple

Tous les exemples utilisent deux petites tables, des clients et des commandes :

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

Ce qui donne :

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

Alice a passé deux commandes, Bob une seule, Chloé et David aucune. La commande 104 appartient à un client 5 qui n'existe pas dans la table des clients : c'est voulu, pour voir comment chaque jointure traite les lignes sans correspondance.

## Un premier merge

On peut écrire `pd.merge(gauche, droite, ...)` ou, ce qui revient au même, `gauche.merge(droite, ...)`. Ici, la clé ne porte pas le même nom dans les deux tables (`client_id` d'un côté, `id_client` de l'autre), on indique donc la colonne à utiliser de chaque côté avec `left_on` et `right_on` :

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

On obtient :

```
   client_id    nom  ville  id_client  commande_id  montant
0          1  Alice   Lyon          1          101     50.0
1          1  Alice   Lyon          1          102     20.0
2          2    Bob  Paris          2          103     99.9
```

Par défaut, `merge` fait une jointure interne (*inner join*) : seules les clés présentes des deux côtés sont gardées, ici les clients 1 et 2. Alice apparaît deux fois, une fois par commande.

Les deux colonnes de clé sont conservées dans le résultat, on peut retirer celle qui ne sert plus avec `.drop(columns="id_client")`. Quand la clé porte le même nom dans les deux tables, c'est plus simple : on passe ce nom à `on`, et la colonne n'apparaît qu'une fois.

```python
cmd = commandes.rename(columns={"id_client": "client_id"})
print(clients.merge(cmd, on="client_id"))
```

```
   client_id    nom  ville  commande_id  montant
0          1  Alice   Lyon          101     50.0
1          1  Alice   Lyon          102     20.0
2          2    Bob  Paris          103     99.9
```

Attention, sans `on` ni `left_on`/`right_on`, pandas joint sur toutes les colonnes que les deux tables ont en commun. Nos tables n'en ont aucune, on obtient donc une `MergeError: No common columns to perform merge on`. Mais si les commandes avaient aussi une colonne `ville` (la ville de livraison), la jointure se ferait sur la ville sans prévenir, et David, qui habite Lyon, récupérerait une commande d'Alice livrée à Lyon. Mieux vaut toujours indiquer la clé.

## Les types de jointures

Le paramètre `how` choisit le type de jointure :

- `inner` (par défaut) : uniquement les clés présentes des deux côtés
- `left` : toutes les lignes de gauche, complétées quand une correspondance existe
- `right` : toutes les lignes de droite
- `outer` : toutes les clés des deux tables
- `cross` : le produit cartésien, chaque ligne de gauche avec chaque ligne de droite
- `left_anti` et `right_anti` : les lignes d'un côté qui n'ont pas de correspondance de l'autre (nouveau dans pandas 3.0, nous y revenons plus bas)

```python
left = pd.merge(clients, commandes, how="left", left_on="client_id", right_on="id_client")
right = pd.merge(clients, commandes, how="right", left_on="client_id", right_on="id_client")
outer = pd.merge(clients, commandes, how="outer", left_on="client_id", right_on="id_client")
```

Avec `how="left"`, tous les clients sont présents. Ceux qui n'ont pas de commande ont des `NaN` dans les colonnes venant de la table de droite :

```
   client_id    nom  ville  id_client  commande_id  montant
0          1  Alice   Lyon        1.0        101.0     50.0
1          1  Alice   Lyon        1.0        102.0     20.0
2          2    Bob  Paris        2.0        103.0     99.9
3          3  Chloé  Lille        NaN          NaN      NaN
4          4  David   Lyon        NaN          NaN      NaN
```

Avec `how="right"`, ce sont toutes les commandes qui sont gardées, y compris la commande 104 du client 5 inconnu :

```
   client_id    nom  ville  id_client  commande_id  montant
0        1.0  Alice   Lyon          1          101     50.0
1        1.0  Alice   Lyon          1          102     20.0
2        2.0    Bob  Paris          2          103     99.9
3        NaN    NaN    NaN          5          104     15.5
```

Et `how="outer"` garde tout le monde :

```
   client_id    nom  ville  id_client  commande_id  montant
0        1.0  Alice   Lyon        1.0        101.0     50.0
1        1.0  Alice   Lyon        1.0        102.0     20.0
2        2.0    Bob  Paris        2.0        103.0     99.9
3        3.0  Chloé  Lille        NaN          NaN      NaN
4        4.0  David   Lyon        NaN          NaN      NaN
5        NaN    NaN    NaN        5.0        104.0     15.5
```

Vous avez sans doute remarqué que les identifiants sont passés de `1` à `1.0` dans les colonnes qui ont reçu des valeurs manquantes. Une colonne `int64` ne peut pas contenir de `NaN`, pandas la convertit donc en `float64`.

Le produit cartésien, lui, n'a pas besoin de clé :

```python
# Sans clé: toutes les combinaisons (seulement si ça a du sens)
cross = pd.merge(clients, commandes, how="cross")
print(cross.shape)
```

```
(16, 6)
```

Quatre clients et quatre commandes donnent 16 lignes. Sur de vraies tables, ça grossit très vite : deux tables de 10 000 lignes produisent 100 millions de lignes.

Enfin, pour associer chaque ligne à la valeur la plus proche plutôt qu'à une valeur égale (rattacher une mesure au dernier tarif en vigueur à sa date, par exemple), pandas propose `pd.merge_asof`. Les deux tables doivent alors être triées sur la clé.

## Joindre sur plusieurs colonnes

Pour joindre sur plusieurs colonnes, on passe des listes à `left_on` et `right_on` (ou à `on`) :

```python
# Exemple artificiel avec une 2e clé
clients2 = clients.assign(pays="FR")
commandes2 = commandes.assign(pays=["FR", "FR", "FR", "FR"])

multi = pd.merge(
    clients2, commandes2,
    how="inner",
    left_on=["client_id", "pays"],
    right_on=["id_client", "pays"],
)
print(multi)
```

```
   client_id    nom  ville pays  id_client  commande_id  montant
0          1  Alice   Lyon   FR          1          101     50.0
1          1  Alice   Lyon   FR          1          102     20.0
2          2    Bob  Paris   FR          2          103     99.9
```

Une ligne n'est associée que si toutes les colonnes de la clé correspondent. Comme `pays` porte le même nom des deux côtés, elle n'apparaît qu'une fois dans le résultat, comme avec `on`.

## Les colonnes en double et `suffixes`

Quand une colonne qui ne fait pas partie de la clé existe dans les deux tables, pandas garde les deux et ajoute un suffixe à leur nom : `_x` pour celle de gauche, `_y` pour celle de droite. Pour l'exemple, on ajoute aux commandes la ville de livraison, dans une colonne qui s'appelle elle aussi `ville` :

```python
commandes_ville = commandes.assign(ville=["Lyon", "Annecy", "Paris", "Nice"])

res = pd.merge(
    clients, commandes_ville,
    how="inner",
    left_on="client_id", right_on="id_client",
    suffixes=("_client", "_livraison"),
)
print(res[["nom", "ville_client", "commande_id", "ville_livraison"]])
```

```
     nom ville_client  commande_id ville_livraison
0  Alice         Lyon          101            Lyon
1  Alice         Lyon          102          Annecy
2    Bob        Paris          103           Paris
```

Sans le paramètre `suffixes`, on aurait eu `ville_x` et `ville_y`, beaucoup moins parlant. Si l'une des deux colonnes ne sert pas, le plus simple est de la retirer avant la jointure : sur de grosses tables, ne garder que les colonnes utiles (`commandes[["id_client", "montant"]]`) allège aussi le résultat.

## Retrouver les lignes sans correspondance

Avant d'exploiter le résultat d'une jointure, il est utile de vérifier ce qui a été associé ou non. Avec `indicator=True`, pandas ajoute une colonne `_merge` qui indique d'où vient chaque ligne : `both`, `left_only` ou `right_only`.

```python
outer_audit = pd.merge(
    clients, commandes,
    how="outer",
    left_on="client_id", right_on="id_client",
    indicator=True,
)
print(outer_audit["_merge"].value_counts())
```

```
_merge
both          3
left_only     2
right_only    1
Name: count, dtype: int64
```

Trois lignes ont trouvé une correspondance, deux clients n'ont pas de commande et une commande n'a pas de client. Pour récupérer les commandes orphelines, on filtre sur cette colonne :

```python
orphelins_cmd = outer_audit[outer_audit["_merge"] == "right_only"]
print(orphelins_cmd)
```

```
   client_id  nom ville  id_client  commande_id  montant      _merge
5        NaN  NaN   NaN        5.0        104.0     15.5  right_only
```

Depuis pandas 3.0, les jointures `left_anti` et `right_anti` donnent directement ces lignes, sans passer par `indicator`. Par exemple, les clients qui n'ont jamais commandé :

```python
sans_commande = pd.merge(clients, commandes, how="left_anti", left_on="client_id", right_on="id_client")
print(sans_commande)
```

```
   client_id    nom  ville  id_client  commande_id  montant
3          3  Chloé  Lille        NaN          NaN      NaN
4          4  David   Lyon        NaN          NaN      NaN
```

## Vérifier la relation avec `validate`

Le paramètre `validate` vérifie le type de relation entre les deux tables avant de faire la jointure. Les valeurs possibles sont :

- `"one_to_one"` ou `"1:1"` : les clés doivent être uniques des deux côtés
- `"one_to_many"` ou `"1:m"` : les clés doivent être uniques à gauche
- `"many_to_one"` ou `"m:1"` : les clés doivent être uniques à droite
- `"many_to_many"` ou `"m:m"` : aucune vérification

Un client peut avoir plusieurs commandes, on attend donc une relation *one-to-many* :

```python
# On s'attend à one-to-many (un client -> plusieurs commandes)
res = pd.merge(
    clients, commandes,
    left_on="client_id", right_on="id_client",
    how="left",
    validate="one_to_many",
)
```

Cette jointure passe sans problème. Avec `validate="one_to_one"`, pandas lève une `MergeError`, car le client 1 apparaît deux fois dans les commandes :

```
pandas.errors.MergeError: Merge keys are not unique in right dataset; not a one-to-one merge
Duplicates in right:
  id_client
         1 ...
```

C'est une vérification peu coûteuse : si une table de référence censée avoir des clés uniques contient des doublons, on le voit tout de suite, au lieu de se retrouver avec des lignes en trop quelques étapes plus loin.

## Joindre sur l'index avec `join`

`merge` peut aussi utiliser l'index comme clé, avec `left_index=True` et `right_index=True` :

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

Les deux donnent le même résultat :

```
             nom  ville  commande_id  montant
client_id                                    
1          Alice   Lyon          101     50.0
1          Alice   Lyon          102     20.0
2            Bob  Paris          103     99.9
```

`DataFrame.join` est un raccourci qui appelle `merge` en interne, avec deux différences à connaître. Sa jointure par défaut est `left` et non `inner`. Et il n'ajoute pas de suffixe par défaut : si les deux tables ont une colonne en commun, il lève l'erreur `columns overlap but no suffix specified` tant qu'on ne lui passe pas `lsuffix` ou `rsuffix`. Avec le paramètre `on`, il joint une colonne de la table de gauche sur l'index de la table de droite.

## Les clés en double

Quand une clé apparaît plusieurs fois des deux côtés (relation *many-to-many*), pandas associe chaque ligne de gauche à chaque ligne de droite qui a la même clé. Si le client 1 a deux adresses dans une table et deux commandes dans l'autre, la jointure produit quatre lignes pour lui. C'est parfois voulu, mais c'est souvent l'explication d'un résultat plus gros que prévu.

Dans ce cas, on dédoublonne avant la jointure (`drop_duplicates(subset=[...])`), ou on agrège. Par exemple, pour avoir le montant total des commandes de chaque client :

```python
# Agréger avant la jointure (ex: montant total par client)
montants = commandes.groupby("id_client", as_index=False)["montant"].sum()
clients_total = clients.merge(montants, left_on="client_id", right_on="id_client", how="left")
print(montants)
print(clients_total)
```

```
   id_client  montant
0          1     70.0
1          2     99.9
2          5     15.5
   client_id    nom  ville  id_client  montant
0          1  Alice   Lyon        1.0     70.0
1          2    Bob  Paris        2.0     99.9
2          3  Chloé  Lille        NaN      NaN
3          4  David   Lyon        NaN      NaN
```

Chaque client n'apparaît plus qu'une fois, avec le total de ses commandes. Le `groupby` est détaillé dans l'article [Python : Comment faire des group by]({% post_url 2025-10-08-Comment-faire-des-group-by-en-python %}).

## Les clés doivent avoir le même type

Un piège classique : la clé est un entier dans une table et une chaîne de caractères dans l'autre, par exemple parce qu'une des sources stocke les identifiants sous forme de texte. pandas refuse alors la jointure :

```python
commandes_str = commandes.astype({"id_client": "str"})
clients.merge(commandes_str, left_on="client_id", right_on="id_client")
```

```
ValueError: You are trying to merge on int64 and str columns for key 'client_id'. If you wish to proceed you should use pd.concat
```

Il suffit de convertir la clé avant la jointure, avec `astype("int64")` ou `pd.to_numeric` :

```python
commandes_str["id_client"] = pd.to_numeric(commandes_str["id_client"])
```

Par contre, un entier et un flottant (`1` et `1.0`) se joignent sans problème.

## Empiler des tables avec `concat`

`merge` et `join` associent des lignes grâce à une clé. `pd.concat` fait autre chose : il colle des tables les unes aux autres. Pour ajouter des lignes :

```python
# concat vertical (ajouter des lignes)
all_clients = pd.concat([clients, pd.DataFrame({"client_id": [6], "nom": ["Emma"], "ville": ["Nice"]})], ignore_index=True)
print(all_clients)
```

```
   client_id    nom  ville
0          1  Alice   Lyon
1          2    Bob  Paris
2          3  Chloé  Lille
3          4  David   Lyon
4          6   Emma   Nice
```

`ignore_index=True` renumérote les lignes, sans quoi Emma garderait l'index 0 de son DataFrame d'origine.

Avec `axis=1`, `concat` met les tables côte à côte en les alignant sur l'index, et garde par défaut les index des deux tables, comme une jointure `outer`. Les index doivent être uniques : avec la table des commandes indexée par `id_client`, où le client 1 apparaît deux fois, on obtient l'erreur `InvalidIndexError: Reindexing only valid with uniquely valued Index objects`. Ça fonctionne en revanche avec les totaux par client :

```python
totaux = commandes.groupby("id_client")["montant"].sum()
horizontal = pd.concat([clients.set_index("client_id"), totaux], axis=1)
print(horizontal)
```

```
     nom  ville  montant
1  Alice   Lyon     70.0
2    Bob  Paris     99.9
3  Chloé  Lille      NaN
4  David   Lyon      NaN
5    NaN    NaN     15.5
```

Voilà, vous savez maintenant combiner deux DataFrame avec pandas et vérifier ce que la jointure a vraiment fait.

## Voir aussi

- [Python : Comment faire des group by]({% post_url 2025-10-08-Comment-faire-des-group-by-en-python %})
- [Automatiser le nettoyage de données avec pandas]({% post_url 2025-12-14-Automatiser-le-nettoyage-de-donnees-avec-pandas %})
- [Python : Comment sauvegarder et charger un dataframe Pandas avec Excel (ou du csv)]({% post_url 2023-12-28-Comment-sauvegarder-un-dataframe-pandas %})
- [Documentation de pandas.merge](https://pandas.pydata.org/docs/reference/api/pandas.merge.html)
- [Guide pandas sur merge, join et concat](https://pandas.pydata.org/docs/user_guide/merging.html)
- [Documentation de pandas.merge_asof](https://pandas.pydata.org/docs/reference/api/pandas.merge_asof.html)
