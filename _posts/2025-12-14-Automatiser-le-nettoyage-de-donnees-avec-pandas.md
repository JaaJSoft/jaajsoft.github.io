---
layout: article
title: "Automatiser le nettoyage de données avec pandas"
description: "Automatiser le nettoyage de données avec pandas : valeurs manquantes, doublons, formats de texte, types, outliers et pipeline de nettoyage réutilisable."
tags:
  - python
  - pandas
  - data
  - nettoyage
author: Pierre Chopinet
---

Les données qu'on récupère d'un export CSV, d'un fichier Excel ou d'une API sont rarement propres : valeurs manquantes, doublons, nombres stockés sous forme de texte, majuscules qui changent d'une ligne à l'autre. Dans ce tutoriel, nous allons voir comment repérer et corriger ces problèmes avec pandas, puis regrouper toutes les étapes dans une fonction qu'on peut relancer à chaque nouvel import.
<!--more-->

Dans cet article :
- Le jeu de données
- Repérer les valeurs manquantes
- Nettoyer le texte
- Corriger les types
- Supprimer les doublons
- Remplir ou supprimer les valeurs manquantes
- Les valeurs aberrantes
- Valider les données
- Une fonction de nettoyage réutilisable
- Journaliser et sauvegarder
- Préparer les données pour un modèle

Pré-requis : Python 3 et pandas (`pip install pandas`), plus scikit-learn pour la dernière section (`pip install scikit-learn`). Les exemples ont été testés avec Python 3.13 et pandas 3.0.5 (pandas 3 demande Python 3.11 ou plus récent).

## Le jeu de données

Pour l'exemple, on part d'une petite table d'employés qui cumule les défauts qu'on rencontre souvent dans un fichier reçu d'une autre équipe :

```python
import pandas as pd
import numpy as np

# Données "sales" typiques
data = {
    "id": [1, 2, 2, 3, 4, 5, 6, 7, None, 9],
    "nom": ["Alice", "Bob", "Bob", "  Charlie ", "diane", "Emma", None, "Frank", "Grace", ""],
    "age": [25, 30, 30, "35", 40, None, 28, "N/A", 22, -5],
    "ville": ["Paris", "paris", "Lyon", "LYON", "Nantes", "Paris", None, "Marseille", "Lyon", "paris"],
    "salaire": [50000, 60000, 60000, "70000", 80000, None, 55000, 90000, 48000, 1000000],
    "date_embauche": ["2020-01-15", "2019-06-20", "2019-06-20", "2021/03/10", None, "2022-01-01", "invalid", "2020-12-01", "2023-05-15", "2024-01-01"],
}

df = pd.DataFrame(data)
print(df)
```

Ce qui donne :

```
    id         nom   age      ville  salaire date_embauche
0  1.0       Alice    25      Paris    50000    2020-01-15
1  2.0         Bob    30      paris    60000    2019-06-20
2  2.0         Bob    30       Lyon    60000    2019-06-20
3  3.0    Charlie     35       LYON    70000    2021/03/10
4  4.0       diane    40     Nantes    80000           NaN
5  5.0        Emma  None      Paris     None    2022-01-01
6  6.0         NaN    28        NaN    55000       invalid
7  7.0       Frank   N/A  Marseille    90000    2020-12-01
8  NaN       Grace    22       Lyon    48000    2023-05-15
9  9.0                -5      paris  1000000    2024-01-01
```

Il y a un peu de tout : des valeurs manquantes sous plusieurs formes (`None`, chaîne vide, `"N/A"`, `"invalid"`), un employé en double (Bob), des villes écrites de plusieurs façons (`Paris` et `paris`, `Lyon` et `LYON`), des espaces autour de `Charlie`, des nombres stockés sous forme de texte (`"35"`, `"70000"`), des valeurs absurdes (un âge de -5 ans, un salaire d'un million) et des dates dans deux formats différents.

Les types des colonnes le confirment :

```python
print(df.dtypes)
```

```
id               float64
nom                  str
age               object
ville                str
salaire           object
date_embauche        str
dtype: object
```

Depuis pandas 3, une colonne qui ne contient que du texte a le type `str` (c'était `object` avant), et un `None` y est converti en `NaN`. Les colonnes `age` et `salaire`, qui mélangent nombres et chaînes, restent en `object`, d'où les `None` dans l'affichage précédent. Quant à `id`, il est passé en `float64` à cause de sa valeur manquante.

## Repérer les valeurs manquantes

`isnull()` (ou son alias `isna()`) indique pour chaque cellule si la valeur manque. En faisant la somme par colonne, on obtient le nombre de valeurs manquantes :

```python
# Compter les NaN par colonne
print(df.isnull().sum())
```

```
id               1
nom              1
age              1
ville            1
salaire          1
date_embauche    1
dtype: int64
```

`df.isnull().mean() * 100` donne la même information en pourcentage, et `df[df.isnull().any(axis=1)]` affiche les lignes qui ont au moins une valeur manquante.

Seuls les vrais `None` sont comptés ici : la chaîne vide, `"N/A"` et `"invalid"` sont des chaînes de caractères comme les autres. Pour trouver ce genre de marqueurs dans une colonne censée être numérique, on peut chercher les valeurs présentes qui ne se convertissent pas en nombre :

```python
# Valeurs présentes mais impossibles à convertir en nombre
mask = pd.to_numeric(df["age"], errors="coerce").isna() & df["age"].notna()
print(df.loc[mask, "age"])
```

```
7    N/A
Name: age, dtype: object
```

On remplace ensuite tous les marqueurs connus par `NaN` :

```python
# Remplacer les marqueurs courants par NaN
df = df.replace(["N/A", "", "NULL", "-", "invalid"], np.nan)
print(df.isnull().sum())
```

```
id               1
nom              2
age              2
ville            1
salaire          1
date_embauche    2
dtype: int64
```

Les compteurs augmentent : le nom vide, le `"N/A"` de l'âge et la date `"invalid"` sont maintenant considérés comme manquants.

## Nettoyer le texte

Les méthodes de l'accesseur `.str` s'appliquent à toute une colonne d'un coup, et laissent les valeurs manquantes telles quelles :

```python
# Supprimer les espaces avant et après
df["nom"] = df["nom"].str.strip()

# Remplacer les espaces multiples à l'intérieur par un seul
df["nom"] = df["nom"].str.replace(r"\s+", " ", regex=True)

# Première lettre en majuscule, le reste en minuscules
df["nom"] = df["nom"].str.capitalize()
df["ville"] = df["ville"].str.strip().str.capitalize()

print(df[["nom", "ville"]])
```

```
       nom      ville
0    Alice      Paris
1      Bob      Paris
2      Bob       Lyon
3  Charlie       Lyon
4    Diane     Nantes
5     Emma      Paris
6      NaN        NaN
7    Frank  Marseille
8    Grace       Lyon
9      NaN      Paris
```

`str.lower()`, `str.upper()` et `str.title()` existent aussi. Attention, `capitalize()` passe en minuscules tout ce qui suit la première lettre : "St-Etienne" devient "St-etienne".

Pour corriger les fautes de frappe et les variantes d'écriture, qu'un changement de casse ne règle pas, on passe un dictionnaire de correspondance à `replace` :

```python
# Corriger les variantes que capitalize() ne règle pas
mapping_ville = {"Marseilles": "Marseille", "Pari": "Paris", "St-etienne": "Saint-Étienne"}
df["ville"] = df["ville"].replace(mapping_ville)
```

Notre jeu de données ne contient aucune de ces variantes, mais c'est typiquement le genre de dictionnaire qui s'enrichit au fil des imports. Les clés doivent correspondre aux valeurs après `capitalize()`, d'où le `"St-etienne"`.

## Corriger les types

`pd.to_numeric` convertit une colonne en nombres. Avec `errors="coerce"`, les valeurs qui ne peuvent pas être converties deviennent `NaN` au lieu de lever une erreur :

```python
# Convertir en numérique (erreurs -> NaN)
df["age"] = pd.to_numeric(df["age"], errors="coerce")
df["salaire"] = pd.to_numeric(df["salaire"], errors="coerce")
```

C'est pratique, mais c'est aussi un bon moyen de perdre des données sans s'en rendre compte, d'où l'intérêt d'avoir cherché les valeurs non convertibles avant.

`pd.to_datetime` fonctionne de la même façon pour les dates. Avec un format strict, la date `"2021/03/10"` de Charlie devient `NaT` (l'équivalent de `NaN` pour les dates), en plus des deux dates déjà manquantes :

```python
dates = pd.to_datetime(df["date_embauche"], errors="coerce", format="%Y-%m-%d")
print(dates.isna().sum())
```

```
3
```

Avec `format="mixed"`, pandas analyse chaque valeur séparément et accepte les deux formats :

```python
# Format mixte (plusieurs formats dans la même colonne)
df["date_embauche"] = pd.to_datetime(df["date_embauche"], errors="coerce", format="mixed")
print(df["date_embauche"])
```

```
0   2020-01-15
1   2019-06-20
2   2019-06-20
3   2021-03-10
4          NaT
5   2022-01-01
6          NaT
7   2020-12-01
8   2023-05-15
9   2024-01-01
Name: date_embauche, dtype: datetime64[us]
```

Les colonnes ont maintenant les bons types :

```python
print(df.dtypes)
```

```
id                      float64
nom                         str
age                     float64
ville                       str
salaire                 float64
date_embauche    datetime64[us]
dtype: object
```

`age` et `salaire` sont en `float64` : une colonne d'entiers qui contient des `NaN` est forcément stockée en flottants. Pour garder des entiers, pandas propose le type `Int64` (avec une majuscule), qui accepte les valeurs manquantes et les affiche `<NA>` :

```python
print(df["age"].astype("Int64").head(6))
```

```
0      25
1      30
2      30
3      35
4      40
5    <NA>
Name: age, dtype: Int64
```

Un simple `astype(int)` échoue dès qu'il reste un `NaN` (`IntCastingNaNError: Cannot convert non-finite values (NA or inf) to integer`).

Enfin, une colonne texte qui ne contient que quelques valeurs différentes, comme `ville`, peut passer en `category` avec `astype("category")` : chaque valeur distincte n'est stockée qu'une fois, ce qui économise de la mémoire sur de gros volumes. Par contre, on ne peut plus y écrire une valeur qui ne fait pas partie des catégories : un `fillna("Inconnu")` lève une `TypeError`. C'est donc une conversion à faire à la fin du nettoyage.

## Supprimer les doublons

`duplicated()` repère les lignes déjà vues, en comparant toutes les colonnes ou seulement certaines :

```python
# Lignes complètement identiques
print(df.duplicated().sum())

# Doublons sur une colonne (keep=False : marque toutes les occurrences)
print(df[df.duplicated(subset=["id"], keep=False)])
```

```
0
    id  nom   age  ville  salaire date_embauche
1  2.0  Bob  30.0  Paris  60000.0    2019-06-20
2  2.0  Bob  30.0   Lyon  60000.0    2019-06-20
```

Aucune ligne n'est entièrement identique, puisque les deux lignes de Bob n'ont pas la même ville. Sur la colonne `id`, en revanche, on les retrouve bien : avec `keep=False`, `duplicated` marque toutes les occurrences, ce qui permet de les afficher avant de décider laquelle garder. Le texte a d'ailleurs été nettoyé avant cette étape, car `Paris` et `paris` suffisent à rendre deux lignes différentes.

```python
# Supprimer les doublons sur l'id (garder la première occurrence)
df = df.drop_duplicates(subset=["id"], keep="first")
```

`keep="last"` garde la dernière occurrence, et `keep=False` supprime toutes les lignes en double. Ici, rien ne dit laquelle des deux villes de Bob est la bonne : c'est le genre de question à poser à ceux qui fournissent les données.

## Remplir ou supprimer les valeurs manquantes

Le plus simple est de supprimer les lignes incomplètes avec `dropna()`. Attention, sans argument, il supprime toutes les lignes qui contiennent au moins un `NaN` : sur notre jeu de données, il ne resterait que 3 lignes sur 9. Mieux vaut préciser ce qu'on veut supprimer :

```python
# Supprimer lignes où toutes les valeurs sont NaN
df_clean = df.dropna(how="all")

# Supprimer lignes où des colonnes spécifiques sont NaN
df_clean = df.dropna(subset=["id", "nom"])

# Supprimer colonnes avec trop de NaN (ex: >50%)
seuil = len(df) * 0.5
df_clean = df.dropna(axis=1, thresh=seuil)
```

`thresh` est le nombre minimum de valeurs non manquantes qu'une colonne doit avoir pour être gardée.

Plutôt que de supprimer des lignes, on peut aussi remplacer les valeurs manquantes (on parle d'imputation) :

```python
# Remplir avec une valeur fixe
df["ville"] = df["ville"].fillna("Inconnu")

# Remplir avec la médiane (colonnes numériques)
df["age"] = df["age"].fillna(df["age"].median())
df["salaire"] = df["salaire"].fillna(df["salaire"].median())
print(df)
```

```
    id      nom   age      ville    salaire date_embauche
0  1.0    Alice  25.0      Paris    50000.0    2020-01-15
1  2.0      Bob  30.0      Paris    60000.0    2019-06-20
3  3.0  Charlie  35.0       Lyon    70000.0    2021-03-10
4  4.0    Diane  40.0     Nantes    80000.0           NaT
5  5.0     Emma  28.0      Paris    65000.0    2022-01-01
6  6.0      NaN  28.0    Inconnu    55000.0           NaT
7  7.0    Frank  28.0  Marseille    90000.0    2020-12-01
8  NaN    Grace  22.0       Lyon    48000.0    2023-05-15
9  9.0      NaN  -5.0      Paris  1000000.0    2024-01-01
```

Emma et Frank reçoivent l'âge médian (28 ans), et Emma le salaire médian (65 000). La médiane convient mieux que la moyenne ici : tirée vers le haut par le salaire d'un million, la moyenne des salaires connus est de 181 625.

Pour des données ordonnées, des mesures dans le temps par exemple, `ffill()` propage la dernière valeur connue, `bfill()` la suivante, et `interpolate()` calcule une valeur intermédiaire. Ces méthodes dépendent de l'ordre des lignes : sur notre table d'employés, `ffill()` aurait donné à Emma le salaire de Diane, la ligne du dessus.

## Les valeurs aberrantes

Une méthode classique pour repérer les valeurs aberrantes est l'écart interquartile (IQR) : on considère comme suspecte toute valeur située à plus d'une fois et demie l'écart interquartile en dessous du premier quartile, ou au-dessus du troisième.

```python
# Identifier les outliers via IQR
def detect_outliers_iqr(df, col):
    Q1 = df[col].quantile(0.25)
    Q3 = df[col].quantile(0.75)
    IQR = Q3 - Q1
    lower = Q1 - 1.5 * IQR
    upper = Q3 + 1.5 * IQR
    return df[(df[col] < lower) | (df[col] > upper)]

# Détecter outliers dans salaire
outliers = detect_outliers_iqr(df, "salaire")
print("Outliers salaire:")
print(outliers[["nom", "salaire"]])
```

```
Outliers salaire:
   nom    salaire
9  NaN  1000000.0
```

Le salaire d'un million est bien repéré. Appliquée à l'âge, la même fonction signale l'âge de -5 ans, mais aussi les 40 ans de Diane, qui n'ont rien d'anormal : sur un échantillon aussi petit, l'écart interquartile est étroit. Une valeur aberrante au sens statistique n'est pas forcément une erreur, c'est à valider avec ceux qui connaissent les données.

Quand on connaît les limites acceptables, des règles métier sont plus fiables :

```python
# Règles métier : âge entre 18 et 70, salaire entre 20k et 200k
df_clean = df[
    (df["age"] >= 18) & (df["age"] <= 70) &
    (df["salaire"] >= 20000) & (df["salaire"] <= 200000)
]
print(len(df_clean))
```

```
8
```

Seule la dernière ligne, avec son âge négatif et son salaire d'un million, est écartée.

Une autre approche consiste à plafonner les valeurs extrêmes plutôt que de supprimer les lignes (on parle de *winsorisation*) :

```python
# Plafonner les valeurs extrêmes (99e percentile)
upper_limit = df["salaire"].quantile(0.99)
df["salaire"] = df["salaire"].clip(upper=upper_limit)
```

Sur 9 lignes, le 99e percentile vaut 927 200, à peine moins que le maximum : cette technique n'a de sens que sur un volume de données plus important.

## Valider les données

Une fois le nettoyage fait, une fonction de validation vérifie que les règles attendues sont bien respectées :

```python
def validate_data(df):
    errors = []

    # ID non nul et unique
    if df["id"].isnull().any():
        errors.append("ID manquants détectés")
    if df["id"].duplicated().any():
        errors.append("ID doublons détectés")

    # Age valide
    if (df["age"] < 0).any() or (df["age"] > 120).any():
        errors.append("Ages invalides détectés")

    # Salaire positif
    if (df["salaire"] < 0).any():
        errors.append("Salaires négatifs détectés")

    # Date dans le futur
    if (df["date_embauche"] > pd.Timestamp.now()).any():
        errors.append("Dates d'embauche dans le futur")

    return errors

errors = validate_data(df)
if errors:
    print("Erreurs de validation:")
    for e in errors:
        print(f"- {e}")
else:
    print("Validation OK")
```

```
Erreurs de validation:
- ID manquants détectés
- Ages invalides détectés
```

Grace n'a toujours pas d'identifiant, et l'âge de -5 ans est encore là, puisque le filtre des règles métier a été appliqué à `df_clean` et non à `df`. Les vérifications peuvent aussi croiser plusieurs colonnes, par exemple pour repérer un salaire incohérent avec l'âge : `df[(df["age"] < 25) & (df["salaire"] > 100000)]`.

Pour avoir une vue d'ensemble, une petite fonction de rapport affiche le nombre de lignes, les valeurs manquantes, les doublons, les types et les statistiques descriptives :

```python
def quality_report(df):
    print("=== Rapport de qualité ===")
    print(f"Lignes totales: {len(df)}")
    print(f"Colonnes: {len(df.columns)}")
    print("\nValeurs manquantes par colonne:")
    print(df.isnull().sum())
    print("\nDoublons (toutes colonnes):", df.duplicated().sum())
    print("\nTypes de données:")
    print(df.dtypes)
    print("\nStatistiques descriptives:")
    print(df.describe(include="all"))

quality_report(df)
```

Avec `include="all"`, `describe` donne à la fois les statistiques des colonnes numériques (moyenne, quartiles...) et celles des colonnes texte (nombre de valeurs distinctes, valeur la plus fréquente).

## Une fonction de nettoyage réutilisable

Toutes ces étapes peuvent être regroupées dans une fonction, qu'on applique directement aux données brutes à chaque nouvel import :

```python
def clean_employee_data(df):
    """Pipeline de nettoyage pour données employés"""
    df = df.copy()  # Ne pas modifier l'original

    # 1) Remplacer marqueurs de valeurs manquantes
    df = df.replace(["N/A", "", "NULL", "-", "invalid", "n/a"], np.nan)

    # 2) Nettoyer les chaînes
    for col in ["nom", "ville"]:
        if col in df.columns:
            df[col] = df[col].str.strip().str.capitalize()

    # 3) Normaliser les villes (corriger les alias/variantes que capitalize() ne gère pas)
    ville_map = {"Marseilles": "Marseille", "Pari": "Paris", "St-etienne": "Saint-Étienne"}
    df["ville"] = df["ville"].replace(ville_map)

    # 4) Convertir les types
    df["id"] = pd.to_numeric(df["id"], errors="coerce")
    df["age"] = pd.to_numeric(df["age"], errors="coerce")
    df["salaire"] = pd.to_numeric(df["salaire"], errors="coerce")
    df["date_embauche"] = pd.to_datetime(df["date_embauche"], errors="coerce", format="mixed")

    # 5) Supprimer doublons (sur id)
    df = df.drop_duplicates(subset=["id"], keep="first")

    # 6) Imputer les valeurs manquantes AVANT de filtrer
    # (sinon le filtre des bornes élimine les NaN et l'imputation ne s'applique jamais)
    df["age"] = df["age"].fillna(df["age"].median())
    df["salaire"] = df["salaire"].fillna(df["salaire"].median())
    df["ville"] = df["ville"].fillna("Inconnu")

    # 7) Filtrer valeurs aberrantes (sur des valeurs désormais imputées)
    df = df[
        (df["age"] >= 18) & (df["age"] <= 70) &
        (df["salaire"] >= 20000) & (df["salaire"] <= 300000)
    ]

    # 8) Supprimer lignes avec ID ou nom manquant
    df = df.dropna(subset=["id", "nom"])

    # 9) Réinitialiser l'index
    df = df.reset_index(drop=True)

    return df

# Appliquer le pipeline sur les données brutes
df_brut = pd.DataFrame(data)
df_clean = clean_employee_data(df_brut)
print(df_clean)
```

```
    id      nom   age      ville  salaire date_embauche
0  1.0    Alice  25.0      Paris  50000.0    2020-01-15
1  2.0      Bob  30.0      Paris  60000.0    2019-06-20
2  3.0  Charlie  35.0       Lyon  70000.0    2021-03-10
3  4.0    Diane  40.0     Nantes  80000.0           NaT
4  5.0     Emma  28.0      Paris  65000.0    2022-01-01
5  7.0    Frank  28.0  Marseille  90000.0    2020-12-01
```

On obtient 6 employés, sans doublon, avec des noms et des villes homogènes. Outre la deuxième ligne de Bob, trois lignes ont été écartées : celle de Grace, qui n'avait pas d'identifiant, celle qui n'avait ni nom ni ville, et celle qui avait un âge négatif. Seule la date d'embauche de Diane reste manquante.

L'ordre des étapes compte. Les doublons sont supprimés avant l'imputation, pour que les lignes en double ne comptent pas deux fois dans la médiane. Et l'imputation se fait avant le filtre des bornes : une comparaison avec `NaN` renvoie toujours `False`, le filtre éliminerait donc les lignes incomplètes avant qu'on ait pu les compléter.

## Journaliser et sauvegarder

Quand le nettoyage tourne automatiquement, il est utile de garder une trace de ce qu'il a fait. Le module `logging` s'en charge, ici sur une version réduite du nettoyage :

```python
import logging

logging.basicConfig(level=logging.INFO, format="%(asctime)s - %(message)s")

def clean_with_logging(df):
    df = df.copy()
    n_initial = len(df)
    logging.info(f"Démarrage nettoyage : {n_initial} lignes")

    # Doublons
    n_duplicates = df.duplicated(subset=["id"]).sum()
    df = df.drop_duplicates(subset=["id"])
    logging.info(f"Doublons supprimés : {n_duplicates}")

    # NaN
    n_before = len(df)
    df = df.dropna(subset=["id", "nom"])
    logging.info(f"Lignes avec NaN critiques supprimées : {n_before - len(df)}")

    logging.info(f"Nettoyage terminé : {len(df)} lignes restantes")
    return df

df_log = clean_with_logging(df_brut)
```

Sur les données brutes, on obtient quelque chose comme :

```
2026-10-07 20:57:23,344 - Démarrage nettoyage : 10 lignes
2026-10-07 20:57:23,345 - Doublons supprimés : 1
2026-10-07 20:57:23,346 - Lignes avec NaN critiques supprimées : 2
2026-10-07 20:57:23,346 - Nettoyage terminé : 7 lignes restantes
```

Enfin, on sauvegarde le résultat :

```python
# CSV
df_clean.to_csv("data_clean.csv", index=False)

# Excel
df_clean.to_excel("data_clean.xlsx", index=False)

# Parquet (garde les types des colonnes)
df_clean.to_parquet("data_clean.parquet", index=False)
```

L'export Excel demande le paquet `openpyxl`, et Parquet le paquet `pyarrow`. Contrairement au CSV, Parquet conserve les types des colonnes : relue avec `pd.read_parquet`, la colonne `date_embauche` est toujours une date, alors qu'avec `pd.read_csv` elle redevient du texte. Les options de `to_csv` et `to_excel` sont détaillées dans l'article [Python : Comment sauvegarder et charger un dataframe Pandas avec Excel (ou du csv)]({% post_url 2023-12-28-Comment-sauvegarder-un-dataframe-pandas %}).

## Préparer les données pour un modèle

Si les données doivent ensuite alimenter un modèle de *machine learning*, deux transformations reviennent souvent. La première met les colonnes numériques à la même échelle, ici avec scikit-learn :

```python
from sklearn.preprocessing import StandardScaler, MinMaxScaler

# Standardisation (moyenne=0, std=1)
scaler = StandardScaler()
df_clean["salaire_scaled"] = scaler.fit_transform(df_clean[["salaire"]])

# Normalisation (0-1)
scaler = MinMaxScaler()
df_clean["age_normalized"] = scaler.fit_transform(df_clean[["age"]])
print(df_clean[["nom", "salaire", "salaire_scaled", "age", "age_normalized"]])
```

```
       nom  salaire  salaire_scaled   age  age_normalized
0    Alice  50000.0       -1.469416  25.0        0.000000
1      Bob  60000.0       -0.702764  30.0        0.333333
2  Charlie  70000.0        0.063888  35.0        0.666667
3    Diane  80000.0        0.830540  40.0        1.000000
4     Emma  65000.0       -0.319438  28.0        0.200000
5    Frank  90000.0        1.597191  28.0        0.200000
```

`StandardScaler` centre les valeurs sur 0 avec un écart-type de 1, `MinMaxScaler` les ramène entre 0 et 1.

La seconde transforme les colonnes texte en nombres. `pd.get_dummies` crée une colonne par valeur (*one-hot encoding*), et les codes d'une colonne `category` donnent un numéro à chaque valeur :

```python
# One-hot encoding
df_encoded = pd.get_dummies(df_clean, columns=["ville"], prefix="ville")
print(df_encoded[["nom", "ville_Lyon", "ville_Paris"]])

# Label encoding
df_clean["ville_code"] = df_clean["ville"].astype("category").cat.codes
print(df_clean[["ville", "ville_code"]])
```

```
       nom  ville_Lyon  ville_Paris
0    Alice       False         True
1      Bob       False         True
2  Charlie        True        False
3    Diane       False        False
4     Emma       False         True
5    Frank       False        False
       ville  ville_code
0      Paris           3
1      Paris           3
2       Lyon           0
3     Nantes           2
4      Paris           3
5  Marseille           1
```

`get_dummies` crée aussi `ville_Marseille` et `ville_Nantes`, qui ne sont pas affichées ici. Ses colonnes contiennent des booléens ; pour avoir des 0 et des 1, on ajoute `dtype=int`. Les codes de catégorie suivent l'ordre alphabétique des villes.

Voilà, vous avez une base de nettoyage à adapter à vos propres données. Le plus long reste de découvrir les défauts d'un nouveau fichier : une fois repérés, il suffit de les ajouter à la fonction de nettoyage.

## Voir aussi

- [Python : Comment merger deux DataFrame pandas]({% post_url 2025-08-31-Comment-merger-deux-dataframe-pandas %})
- [Python : Comment faire des group by]({% post_url 2025-10-08-Comment-faire-des-group-by-en-python %})
- [Python : Comment sauvegarder et charger un dataframe Pandas avec Excel (ou du csv)]({% post_url 2023-12-28-Comment-sauvegarder-un-dataframe-pandas %})
- [Guide pandas sur les valeurs manquantes](https://pandas.pydata.org/docs/user_guide/missing_data.html)
- [Guide pandas sur les données texte](https://pandas.pydata.org/docs/user_guide/text.html)
- [Documentation officielle de pandas](https://pandas.pydata.org/docs/)
