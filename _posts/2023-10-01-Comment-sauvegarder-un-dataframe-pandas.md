---
layout: article
title: "Python : Comment sauvegarder des dataframes Pandas"
author: Pierre Chopinet
tags:

- python
- pandas
- dataframe
- persistence
- tutoriel
- excel
- csv

---

Dans ce tutoriel, nous allons apprendre à sauvegarder et charger des _dataframes_ Pandas en csv ou excel, afin
ajouter de la persistence à vos applications python ou _notebooks_ jupyter. <!--more-->

## Notre premiere persistence

Pour notre première persistence, nous allons utiliser le format de fichier `csv`. Format texte simple, mais très
pratique pour de la _datascience_.

Dans l'intégralité de ce guide, nous allons utiliser un dataframe de test nommé 'df' que nous initialisons de la façon
suivante :

```python
df = pd.DataFrame({
    "A": [1, 2, 3, 4],
    "B": [6, 7, 8, 9],
    "C": [10, 11, 12, 13],
    "D": [14, 15, 16, 17]
})
```

Ce qui donne le dataframe suivant :

```
   A  B   C   D
0  1  6  10  14
1  2  7  11  15
2  3  8  12  16
3  4  9  13  17
```

### Sauvegarde des données vers un csv



Le _delimiter_ est le caractère au sein du fichier qui sépare nos valeurs. Les
délimiteurs courants sont `,` et `;`.
...
Notre fichier csv contient :

```
6.000000000000000000e+00,9.000000000000000000e+00,4.200000000000000000e+01
4.000000000000000000e+00,2.000000000000000000e+00,9.000000000000000000e+00
```

Les nombres sont enregistrés par défaut en nombre flottant, pour régler ça, on
spécifie un format à notre export :

```python

```

Finalement, on obtient un format en entier :

```
6,9,42
4,2,9
```

Par défaut, l'index du dataframe est exporté. Ce qu'on ne veut pas forcément

### Chargement des données depuis un csv

...

```python

```

Si on a pas de header dans le fichier

Et voilà, on a notre première persistence de données avec pandas !

## Persistence au format excel

Excel est utilisé partout de nos jours, il peut être pratique d'extraire nos données de notre dataframe au format excel
afin de le partager à d'autres équipes.
Ou à l'inverse d'autres équipes non techniques peuvent nous fournir des données au format excel

### Sauvegarde des données vers un excel

On utilise une autre méthode pandas `to_excel` :

```python

```

### Chargement des données depuis un excel

```python


```

```
[[ 6.  9. 42.]
 [ 4.  2.  9.]]
```

## Conclusion

Voilà, vous êtes maintenant capable de faire persister vos données pandas en excel ou en csv.

## Voir aussi

- [La documentation de Pandas](https://pandas.pydata.org/docs/)
- [Comment sauvegarder des tableaux NumPy](https://blog.jaaj.dev/2022/01/25/Comment-sauvegarder-un-tableau-numpy.html)
- [Comment faire des requêtes HTTP en python avec requests](https://blog.jaaj.dev/2020/05/22/Comment-faire-des-requetes-http-en-python-avec-requests.html)
