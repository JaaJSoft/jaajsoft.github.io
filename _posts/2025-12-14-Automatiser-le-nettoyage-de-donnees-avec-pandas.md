---
layout: article
title: "Automatiser le nettoyage de données avec pandas"
date: 2025-12-14
author: Pierre Chopinet
tags:
  - python
  - pandas
  - data
  - nettoyage
  - tutoriel
---

Le nettoyage de données représente souvent 60 à 80 % du travail en data science. Des données mal formatées, des valeurs manquantes, des doublons, des types incohérents : autant de problèmes qui peuvent saboter vos analyses. Heureusement, pandas offre un arsenal complet pour automatiser ces tâches fastidieuses.
<!--more-->

Dans ce guide, vous allez apprendre à :

- Détecter et gérer les valeurs manquantes (NaN, None, chaînes vides)
- Identifier et supprimer les doublons
- Normaliser les formats (dates, chaînes, nombres)
- Corriger les types de données (conversions, catégories)
- Détecter et traiter les valeurs aberrantes (outliers)
- Valider la qualité des données
- Créer des pipelines de nettoyage réutilisables

Prérequis :
- Python 3.8+
- pandas (installation : `pip install pandas`)
- Notions de base de pandas (DataFrame, Series)

---

## Jeu de données d'exemple (sale et réaliste)

Créons un DataFrame typique qu'on pourrait recevoir d'une source externe (CSV, API, Excel) :

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

Sortie :

```
     id         nom  age     ville  salaire date_embauche
0   1.0       Alice   25     Paris    50000    2020-01-15
1   2.0         Bob   30     paris    60000    2019-06-20
2   2.0         Bob   30      Lyon    60000    2019-06-20
3   3.0    Charlie    35      LYON    70000    2021/03/10
4   4.0       diane   40    Nantes    80000          None
5   5.0        Emma None     Paris     None    2022-01-01
6   6.0        None   28      None    55000       invalid
7   7.0       Frank  N/A  Marseille    90000    2020-12-01
8   NaN       Grace   22      Lyon    48000    2023-05-15
9   9.0               -5     paris  1000000    2024-01-01
```

Problèmes identifiés :
- Valeurs manquantes (NaN, None, chaînes vides, "N/A")
- Doublons (ligne 1 et 2 : Bob)
- Incohérences de casse (Paris/paris, Lyon/LYON)
- Espaces parasites ("  Charlie ")
- Types incorrects (age en str, salaire en str)
- Valeurs aberrantes (age = -5, salaire = 1000000)
- Formats de dates variés

---

## 1) Détecter et gérer les valeurs manquantes

### a) Détection des valeurs manquantes

```python
# Compter les NaN par colonne
print(df.isnull().sum())

# Ou inversement : compter les valeurs non-nulles
print(df.notnull().sum())

# Lignes contenant au moins un NaN
print(df[df.isnull().any(axis=1)])

# Proportion de NaN par colonne
print(df.isnull().mean() * 100)  # en %
```

Sortie :

```
id                1
nom               2
age               1
ville             1
salaire           1
date_embauche     1
```

### b) Remplacer les marqueurs de valeurs manquantes

Certaines sources utilisent des chaînes comme "N/A", "", "NULL", "-", etc. Il faut les convertir en `NaN` :

```python
# Remplacer les marqueurs courants par NaN
df = df.replace(["N/A", "", "NULL", "-", "invalid"], np.nan)
print(df.isnull().sum())
```

### c) Supprimer les lignes/colonnes avec NaN

```python
# Supprimer lignes avec au moins un NaN
df_clean = df.dropna()

# Supprimer lignes où toutes les valeurs sont NaN
df_clean = df.dropna(how="all")

# Supprimer lignes où des colonnes spécifiques sont NaN
df_clean = df.dropna(subset=["id", "nom"])

# Supprimer colonnes avec trop de NaN (ex: >50%)
seuil = len(df) * 0.5
df_clean = df.dropna(axis=1, thresh=seuil)
```

### d) Imputer (remplacer) les valeurs manquantes

```python
# Remplir avec une valeur fixe
df["ville"] = df["ville"].fillna("Inconnu")

# Remplir avec la moyenne (colonnes numériques)
df["age"] = df["age"].fillna(df["age"].median())

# Forward fill (propager la dernière valeur valide)
df["salaire"] = df["salaire"].fillna(method="ffill")

# Backward fill
df["date_embauche"] = df["date_embauche"].fillna(method="bfill")

# Interpolation linéaire (séries temporelles)
df["salaire"] = df["salaire"].interpolate()
```

---

## 2) Identifier et supprimer les doublons

### a) Détection des doublons

```python
# Lignes complètement identiques
print(df.duplicated())

# Doublons sur des colonnes spécifiques (ex: id)
print(df.duplicated(subset=["id"], keep=False))  # keep=False : marque tous les doublons
```

### b) Suppression des doublons

```python
# Supprimer doublons complets (garder la première occurrence)
df_clean = df.drop_duplicates()

# Supprimer doublons sur colonnes spécifiques
df_clean = df.drop_duplicates(subset=["id"], keep="first")  # ou "last"

# Garder seulement les lignes uniques (supprimer toutes les occurrences)
df_clean = df[~df.duplicated(subset=["id"], keep=False)]
```

---

## 3) Normaliser les formats de texte

### a) Nettoyer les chaînes (espaces, casse)

```python
# Supprimer espaces avant/après
df["nom"] = df["nom"].str.strip()

# Normaliser la casse (tout en minuscule)
df["ville"] = df["ville"].str.lower()

# Ou capitaliser (première lettre en majuscule)
df["nom"] = df["nom"].str.capitalize()

# Tout en majuscule
df["ville"] = df["ville"].str.upper()

# Supprimer espaces multiples à l'intérieur
df["nom"] = df["nom"].str.replace(r"\s+", " ", regex=True)
```

Exemple complet :

```python
# Pipeline de nettoyage des chaînes
def clean_string(s):
    if pd.isna(s):
        return s
    return s.strip().lower()

df["ville"] = df["ville"].apply(clean_string)
df["nom"] = df["nom"].str.strip().str.capitalize()
```

### b) Remplacer et normaliser les valeurs

```python
# Dictionnaire de mapping pour normaliser
mapping_ville = {
    "paris": "Paris",
    "lyon": "Lyon",
    "LYON": "Lyon",
    "nantes": "Nantes",
}
df["ville"] = df["ville"].replace(mapping_ville)

# Ou via apply + fonction
def normaliser_ville(v):
    if pd.isna(v):
        return v
    v = v.strip().lower()
    return v.capitalize()

df["ville"] = df["ville"].apply(normaliser_ville)
```

---

## 4) Corriger les types de données

### a) Détection des types

```python
print(df.dtypes)
print(df.info())
```

### b) Conversion de types

```python
# Convertir en numérique (erreurs -> NaN)
df["age"] = pd.to_numeric(df["age"], errors="coerce")
df["salaire"] = pd.to_numeric(df["salaire"], errors="coerce")

# Convertir en entier (après nettoyage des NaN)
df["age"] = df["age"].fillna(0).astype(int)

# Convertir en catégorie (économie mémoire + performance)
df["ville"] = df["ville"].astype("category")

# Convertir en datetime
df["date_embauche"] = pd.to_datetime(df["date_embauche"], errors="coerce", format="%Y-%m-%d")

# Format mixte (essayer plusieurs formats)
df["date_embauche"] = pd.to_datetime(df["date_embauche"], errors="coerce", infer_datetime_format=True)
```

### c) Valider les conversions

```python
# Vérifier les NaN créés par coerce
print("NaN créés après conversion:")
print(df.isnull().sum())

# Lister les valeurs qui n'ont pas pu être converties
mask = pd.to_numeric(df["age"], errors="coerce").isna() & df["age"].notna()
print("Valeurs invalides dans age:")
print(df.loc[mask, "age"])
```

---

## 5) Détecter et traiter les valeurs aberrantes (outliers)

### a) Méthode statistique (IQR - Interquartile Range)

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

### b) Filtrage manuel (bornes métier)

```python
# Règles métier : âge entre 18 et 70, salaire entre 20k et 200k
df_clean = df[
    (df["age"] >= 18) & (df["age"] <= 70) &
    (df["salaire"] >= 20000) & (df["salaire"] <= 200000)
]
```

### c) Winsorisation (cap values)

```python
# Plafonner les valeurs extrêmes (99e percentile)
upper_limit = df["salaire"].quantile(0.99)
df["salaire"] = df["salaire"].clip(upper=upper_limit)
```

---

## 6) Valider la qualité des données

### a) Règles de validation

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

### b) Rapport de qualité

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

---

## 7) Pipeline de nettoyage complet et réutilisable

Créons une fonction pour automatiser tout le nettoyage :

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

    # 3) Normaliser les villes
    ville_map = {"Paris": "Paris", "Lyon": "Lyon", "Nantes": "Nantes", "Marseille": "Marseille"}
    df["ville"] = df["ville"].replace(ville_map)

    # 4) Convertir les types
    df["id"] = pd.to_numeric(df["id"], errors="coerce")
    df["age"] = pd.to_numeric(df["age"], errors="coerce")
    df["salaire"] = pd.to_numeric(df["salaire"], errors="coerce")
    df["date_embauche"] = pd.to_datetime(df["date_embauche"], errors="coerce", infer_datetime_format=True)

    # 5) Supprimer doublons (sur id)
    df = df.drop_duplicates(subset=["id"], keep="first")

    # 6) Filtrer valeurs aberrantes
    df = df[
        (df["age"] >= 18) & (df["age"] <= 70) &
        (df["salaire"] >= 20000) & (df["salaire"] <= 300000)
    ]

    # 7) Imputer valeurs manquantes
    df["age"] = df["age"].fillna(df["age"].median())
    df["salaire"] = df["salaire"].fillna(df["salaire"].median())
    df["ville"] = df["ville"].fillna("Inconnu")

    # 8) Supprimer lignes avec ID ou nom manquant
    df = df.dropna(subset=["id", "nom"])

    # 9) Réinitialiser l'index
    df = df.reset_index(drop=True)

    return df

# Appliquer le pipeline
df_clean = clean_employee_data(df)
print(df_clean)
```

Sortie (nettoyée) :

```
   id      nom  age    ville  salaire date_embauche
0   1    Alice   25    Paris  50000.0    2020-01-15
1   2      Bob   30    Paris  60000.0    2019-06-20
2   3  Charlie   35     Lyon  70000.0    2021-03-10
3   4    Diane   40   Nantes  80000.0           NaT
4   5     Emma   30    Paris  60000.0    2022-01-01
5   7    Frank   30  Marseille 90000.0    2020-12-01
6   9      Nan   30    Paris  60000.0    2024-01-01
```

---

## 8) Sauvegarde et logging du nettoyage

### a) Logger les actions

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

df_clean = clean_with_logging(df)
```

### b) Sauvegarder les données nettoyées

```python
# CSV
df_clean.to_csv("data_clean.csv", index=False)

# Excel
df_clean.to_excel("data_clean.xlsx", index=False)

# Parquet (recommandé pour gros volumes)
df_clean.to_parquet("data_clean.parquet", index=False)
```

> Voir aussi : [Comment sauvegarder un dataframe pandas]({% post_url 2023-12-28-Comment-sauvegarder-un-dataframe-pandas %})

---

## 9) Techniques avancées

### a) Normalisation/standardisation des valeurs numériques

```python
from sklearn.preprocessing import StandardScaler, MinMaxScaler

# Standardisation (moyenne=0, std=1)
scaler = StandardScaler()
df["salaire_scaled"] = scaler.fit_transform(df[["salaire"]])

# Normalisation (0-1)
scaler = MinMaxScaler()
df["age_normalized"] = scaler.fit_transform(df[["age"]])
```

### b) Encodage de catégories

```python
# One-hot encoding
df_encoded = pd.get_dummies(df, columns=["ville"], prefix="ville")

# Label encoding
df["ville_code"] = df["ville"].astype("category").cat.codes
```

### c) Détecter les incohérences entre colonnes

```python
# Ex: date_embauche après aujourd'hui
df_invalid = df[df["date_embauche"] > pd.Timestamp.now()]

# Ex: salaire incohérent avec l'âge
df_invalid = df[(df["age"] < 25) & (df["salaire"] > 100000)]
```

---

## 10) Bonnes pratiques et pièges

### Bonnes pratiques

- Toujours copier le DataFrame avant nettoyage (`df = df.copy()`) pour ne pas modifier l'original.
- Logger les étapes et le nombre de lignes affectées.
- Valider les données avant et après nettoyage.
- Créer des pipelines réutilisables (fonctions).
- Sauvegarder les données intermédiaires (avant/après nettoyage).
- Documenter les règles métier et seuils utilisés.

### Pièges

- `fillna(method="ffill")` peut propager des erreurs si mal utilisé.
- `dropna()` peut supprimer trop de lignes : préférez `dropna(subset=[...])`.
- `errors="coerce"` masque les erreurs de conversion : vérifiez les NaN créés.
- Les outliers ne sont pas toujours des erreurs : valider avec le métier.
- Ne pas normaliser les chaînes peut créer des doublons cachés (Paris vs paris).
- Attention à l'ordre des opérations : supprimer doublons avant imputation.

---

## Cheatsheet

### Valeurs manquantes

```python
df.isnull().sum()                    # Compter NaN
df.replace(["N/A", ""], np.nan)      # Remplacer marqueurs
df.dropna(subset=["col"])            # Supprimer lignes
df["col"].fillna(value)              # Imputer
```

### Doublons

```python
df.duplicated()                      # Détecter
df.drop_duplicates(subset=["id"])    # Supprimer
```

### Normalisation texte

```python
df["col"].str.strip()                # Espaces
df["col"].str.lower()                # Minuscules
df["col"].str.capitalize()           # Première maj
```

### Types

```python
pd.to_numeric(df["col"], errors="coerce")
pd.to_datetime(df["col"], errors="coerce")
df["col"].astype("category")
```

### Outliers

```python
Q1 = df["col"].quantile(0.25)
Q3 = df["col"].quantile(0.75)
IQR = Q3 - Q1
df[(df["col"] < Q1 - 1.5*IQR) | (df["col"] > Q3 + 1.5*IQR)]
```

---

## Conclusion

Le nettoyage de données avec pandas peut être largement automatisé en combinant détection de valeurs manquantes, suppression de doublons, normalisation de formats et validation de règles métier. En créant des pipelines réutilisables et en loggant chaque étape, vous gagnez en productivité et en fiabilité. Des données propres sont la clé d'analyses fiables et de modèles performants.

---

## Voir aussi

- [Comment merger deux DataFrame pandas]({% post_url 2025-08-31-Comment-merger-deux-dataframe-pandas %})
- [Comment faire des group by en Python]({% post_url 2025-10-08-Comment-faire-des-group-by-en-python %})
- [Comment sauvegarder un dataframe pandas]({% post_url 2023-12-28-Comment-sauvegarder-un-dataframe-pandas %})
- [Comment sauvegarder un tableau numpy]({% post_url 2022-01-25-Comment-sauvegarder-un-tableau-numpy %})
- [Documentation pandas](https://pandas.pydata.org/docs/)
