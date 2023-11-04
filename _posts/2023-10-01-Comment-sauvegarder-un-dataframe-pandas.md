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

---

Dans ce tutoriel, nous allons apprendre à sauvegarder et charger des _dataframes_ Pandas en csv ou excel, afin
ajouter de la persistence à vos applications python ou _notebooks_ jupyter. <!--more-->

## Notre premiere persistence

Pour notre première persistence, nous allons utiliser le format de fichier `csv`. Format texte simple, mais très
pratique pour de la _datascience_.

### Sauvegarde des données

...
```python

```
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

### Chargement des données

...
```python

```

Et voilà, on a notre première persistence de données avec pandas !

## Persistence au format excel

Excel est utilisé partout de nos jours, il peut être pratique d'extraire nos données de notre dataframe au format excel
afin de le partager à d'autres équipes.
Ou à l'inverse d'autres équipes non techniques peuvent nous fournir des données au format excel

### Sauvegarde des données

On utilise une autre méthode pandas `to_excel` :

```python

```

### Chargement des données


```python


```

```
[[ 6.  9. 42.]
 [ 4.  2.  9.]]
```

## Conclusion

Voilà, vous êtes maintenant capable de faire persister vos données pandas en excel ou en csv.

## Voir aussi

- [La doc de Pandas](https://pandas.pydata.org/docs/)
- [Comment sauvegarder des tableaux NumPy](https://blog.jaaj.dev/2022/01/25/Comment-sauvegarder-un-tableau-numpy.html)
- [Comment faire des requêtes HTTP en python avec requests](https://blog.jaaj.dev/2020/05/22/Comment-faire-des-requetes-http-en-python-avec-requests.html)
