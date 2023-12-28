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

Pour notre première persistence, nous allons utiliser le format de fichier `.csv`. Format texte simple, mais très
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

Pour exporter vers un csv, on utilise la méthode `to_csv` sur le _dataframe_ :

```python
df.to_csv(
    "export.csv", 
    sep=';'
)
```

Le `sep` est le caractère au sein du fichier qui sépare nos valeurs. Les
délimiteurs courants sont `,` (défaut) et `;` (standard en France).

Notre fichier csv exporté contient :

```csv
;A;B;C;D
0;1;6;10;14
1;2;7;11;15
2;3;8;12;16
3;4;9;13;17
```

Comme on peut le voir, par défaut l'index du dataframe est exporté. Ce qui n'est pas forcément le comportement voulu...

Pour régler ça on passe l'option `index=False` à la méthode :

```python
df.to_csv(
     "export_without_index.csv",
     sep=';',
     index=False
)
```

Ce qui donne le résultat voulu, sans l'index de notre _dataframe_ :

```csv
A;B;C;D
1;6;10;14
2;7;11;15
3;8;12;16
4;9;13;17
```

On peut de la même façon ne pas exporter le _header_ de notre _dataframe_ en passant l'option `header=False` à la
méthode :

```python
df.to_csv(
     "export_without_header.csv",
     sep=';', 
     index=False,
     header=False
)
```

Sans index ni header :

```csv
1;6;10;14
2;7;11;15
3;8;12;16
4;9;13;17
```

Il est aussi possible de choisir quelles colonnes exporter avec l'aide du paramètre `columns=` :

```python
df.to_csv(
    "export_columns.csv",
    sep=';',
    index=False,
    columns=["A", "C"]
)
```

### Chargement des données depuis un csv

Maintenant qu'on sait exporter nos données vers un fichier csv.
Comment charger les données depuis ce même format ?

Pour cela, nous allons utiliser la fonction `read_csv` de pandas :

```python
new_df = pd.read_csv("export_without_index.csv", sep=';')
print(new_df.to_string())
```

On passe comme pour l'export le format de notre séparateur.

```
   A  B   C   D
0  1  6  10  14
1  2  7  11  15
2  3  8  12  16
3  4  9  13  17
```

Cependant, comme nous l'avons vu précédemment, il est possible d'avoir des fichiers `csv` sans _headers_ !
Pour régler ça, on peut passer en paramètre de la fonction _pandas_ les noms des colonnes :

```python
new_df = pd.read_csv(
     "export_without_header.csv",
     sep='3',
     header=None,
     names=['A', 'B', 'C', 'D']
)
print(new_df.to_string())
```

Et voilà, pandas charge correctement le dataframe en utilisant les noms des colonnes que nous avons définies :

```
   A  B   C   D
0  1  6  10  14
1  2  7  11  15
2  3  8  12  16
3  4  9  13  17
```

Et voilà, on a notre première persistence de données avec pandas !

## Persistence au format excel

Excel est utilisé partout de nos jours, il peut être pratique d'extraire nos données de notre dataframe au format excel
afin de le partager à d'autres équipes.
Ou à l'inverse d'autres équipes non techniques peuvent nous fournir des données au format excel

### Pré-requis

Afin d'utiliser excel avec pandas, il est nécessaire d'installer le paquet `openpyxl` :

```Bash
pip3 install openpyxl # (ou python3 -m pip)
```

### Sauvegarde des données vers un excel

On utilise une autre méthode pandas sur notre _dataframe_ nommée `to_excel` :

```python
df.to_excel(
    "export.xlsx",
    sheet_name="export",
    index=False
)
```

Toutes les options présentées précédemment avec le format csv sont aussi disponibles : `header=`, `index=`, etc. ! 
(Sauf le séparateur `sep=` qui est spécifique au format csv)

### Chargement des données depuis un excel

Pour charger des données depuis un excel comme avec un csv, on utilise une fonction de pandas `read_excel`.

```python
excel_df = pd.read_excel(
    "export.xlsx",
    sheet_name="export" # Pour spécifier depuis quelle feuille
)
print(excel_df.to_string())
```

Le paramètre `sheet_name` permet de spécifier depuis quelle feuille lire les données.

Ce qui donne bien le résultat attendu :

```
   A  B   C   D
0  1  6  10  14
1  2  7  11  15
2  3  8  12  16
3  4  9  13  17
```

## Conclusion

Voilà, vous êtes maintenant capable de sauvegarder et charger vos _dataframes_ pandas avec excel ou des fichiers csv.

## Voir aussi

- [La documentation de Pandas](https://pandas.pydata.org/docs/)
- [Comment sauvegarder des tableaux NumPy](https://blog.jaaj.dev/2022/01/25/Comment-sauvegarder-un-tableau-numpy.html)
- [Comment faire des requêtes HTTP en python avec requests](https://blog.jaaj.dev/2020/05/22/Comment-faire-des-requetes-http-en-python-avec-requests.html)
