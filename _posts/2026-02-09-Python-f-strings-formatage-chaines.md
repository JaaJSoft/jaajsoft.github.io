---
layout: article
title: "Python : Comment utiliser les f-string"
author: Pierre Chopinet
tags:
  - python
  - strings
  - formatage
---

Les f-strings sont la façon la plus moderne et lisible de formater des chaînes en Python (depuis Python 3.6). Plus concises que `format()` et plus puissantes que `%`, elles permettent d'intégrer des expressions Python directement dans les chaînes avec une syntaxe simple et élégante.
<!--more-->

Dans cet article :
- Syntaxe de base des f-strings
- Expressions et appels de fonctions
- Formatage des nombres (précision, padding, séparateurs)
- Formatage des dates
- Alignement et largeur
- Échappement des accolades
- Comparaison avec `format()` et `%`
- Astuces et bonnes pratiques

---

## 1) Syntaxe de base

### Intégrer des variables

```python
nom = "Alice"
age = 30

# Ancienne méthode avec %
message = "Bonjour %s, vous avez %d ans." % (nom, age)

# Méthode .format()
message = "Bonjour {}, vous avez {} ans.".format(nom, age)

# f-string (moderne et lisible)
message = f"Bonjour {nom}, vous avez {age} ans."
print(message)  # Bonjour Alice, vous avez 30 ans.
```

**Avantages** :
- ✅ Plus court et plus lisible
- ✅ Les variables sont insérées directement
- ✅ Pas besoin de répéter les variables à la fin

---

## 2) Expressions dans les f-strings

Vous pouvez évaluer des expressions Python directement dans les accolades :

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

### Appeler des fonctions

```python
def prix_ttc(prix_ht, tva=0.20):
    return prix_ht * (1 + tva)

prix = 100
print(f"Prix TTC : {prix_ttc(prix):.2f}€")
# Prix TTC : 120.00€
```

### Accéder aux attributs et méthodes

```python
texte = "python"
print(f"Majuscule : {texte.upper()}")
# Majuscule : PYTHON

print(f"Longueur : {len(texte)} caractères")
# Longueur : 6 caractères
```

---

## 3) Formatage des nombres

Les f-strings offrent une syntaxe puissante pour formater les nombres : précision décimale, séparateurs de milliers, pourcentages, notation scientifique et bases numériques.

### Arrondir à n décimales

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
print(f"{nombre: .2f}")     # 1 234 567.89 (espace)
```

**Pour le format français** (espace comme séparateur) :

```python
import locale
locale.setlocale(locale.LC_ALL, 'fr_FR.UTF-8')

print(f"{nombre:n}")  # 1 234 567,89
```

### Pourcentages

```python
taux = 0.1547

print(f"Taux : {taux:.2%}")  # Taux : 15.47%
print(f"Taux : {taux:.1%}")  # Taux : 15.5%
```

### Notation scientifique

```python
grand_nombre = 1234567890

print(f"{grand_nombre:e}")   # 1.234568e+09
print(f"{grand_nombre:.2e}") # 1.23e+09
```

### Nombres binaires, hexadécimaux, octaux

```python
nombre = 42

print(f"Binaire : {nombre:b}")    # 101010
print(f"Octal : {nombre:o}")      # 52
print(f"Hexadécimal : {nombre:x}")  # 2a
print(f"Hexadécimal (MAJ) : {nombre:X}")  # 2A
print(f"Hex avec préfixe : {nombre:#x}")   # 0x2a
```

---

## 4) Largeur, alignement et padding

Pour créer des tableaux ou aligner du texte, les f-strings permettent de contrôler la largeur, l'alignement et le caractère de remplissage.

### Largeur minimale

```python
nom = "Alice"
age = 30

print(f"{nom:10} | {age:3}")
# Alice      |  30
```

### Alignement

```python
texte = "Python"

print(f"{texte:<10}")  # Python     (gauche)
print(f"{texte:>10}")  #     Python (droite)
print(f"{texte:^10}")  #   Python   (centré)
```

### Padding avec un caractère personnalisé

```python
nombre = 42

print(f"{nombre:0>5}")  # 00042 (zéros à gauche)
print(f"{nombre:*<5}")  # 42*** (étoiles à droite)
print(f"{nombre:-^7}")  # --42--- (tirets centré)
```

### Combiner largeur et précision

```python
prix = 12.5

print(f"{prix:10.2f}")   #      12.50 (10 caractères, 2 décimales)
print(f"{prix:0>10.2f}") # 0000012.50 (padding zéros)
```

---

## 5) Formatage des dates

Les objets `datetime` peuvent être formatés directement dans les f-strings en utilisant les codes de format strftime.

```python
from datetime import datetime

maintenant = datetime.now()

print(f"Date : {maintenant:%Y-%m-%d}")
# Date : 2026-02-09

print(f"Heure : {maintenant:%H:%M:%S}")
# Heure : 14:30:45

print(f"Date complète : {maintenant:%A %d %B %Y}")
# Date complète : Monday 09 February 2026

print(f"Format court : {maintenant:%d/%m/%Y %H:%M}")
# Format court : 09/02/2026 14:30
```

**Format français** :

```python
import locale
locale.setlocale(locale.LC_TIME, 'fr_FR.UTF-8')

print(f"{maintenant:%A %d %B %Y}")
# lundi 09 février 2026
```

---

## 6) Échappement des accolades

Pour afficher des accolades littérales dans une f-string, doublez-les :

```python
x = 10

print(f"La variable x vaut {x}")
# La variable x vaut 10

print(f"Pour afficher {{x}}, écrivez {{{{x}}}}")
# Pour afficher {x}, écrivez {{x}}
```

**Règle** :
- `{variable}` → évalue la variable
- `{{` → affiche `{`
- `}}` → affiche `}`

---

## 7) F-strings multiligne

Les f-strings fonctionnent avec les chaînes multiligne (triple guillemets), pratique pour générer des messages longs ou des templates.

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

---

## 8) Debugging avec f-strings (Python 3.8+)

La syntaxe `{variable=}` affiche le nom de la variable et sa valeur :

```python
x = 10
y = 20

print(f"{x=}, {y=}, {x + y=}")
# x=10, y=20, x + y=30
```

**Très pratique pour le debugging** :

```python
def calculer_total(prix, quantite):
    total = prix * quantite
    print(f"{prix=}, {quantite=}, {total=}")
    return total

calculer_total(12.5, 3)
# prix=12.5, quantite=3, total=37.5
```

---

## 9) F-strings imbriquées

Pour des cas avancés, vous pouvez imbriquer des f-strings pour construire des formats dynamiques :

```python
nombre = 1234.5678
precision = 2

print(f"{nombre:.{precision}f}")
# 1234.57
```

**Exemple avec alignement dynamique** :

```python
texte = "Python"
largeur = 15
alignement = "^"  # centré

print(f"{texte:{alignement}{largeur}}")
#     Python
```

---

## 10) Comparaison avec les autres méthodes

Avant Python 3.6, deux méthodes principales existaient pour formater des chaînes. Voici pourquoi les f-strings sont supérieures.

### Ancien style avec %

```python
nom = "Alice"
age = 30

message = "Bonjour %s, vous avez %d ans." % (nom, age)
```

**Inconvénients** :
- ❌ Syntaxe peu lisible
- ❌ Types à spécifier (`%s`, `%d`, `%f`)
- ❌ Erreurs si le nombre d'arguments ne correspond pas

### Méthode .format()

```python
message = "Bonjour {}, vous avez {} ans.".format(nom, age)

# Avec indices
message = "Bonjour {0}, vous avez {1} ans. {0} est génial !".format(nom, age)

# Avec noms
message = "Bonjour {nom}, vous avez {age} ans.".format(nom=nom, age=age)
```

**Inconvénients** :
- ❌ Verbeux (répétition des variables)
- ❌ Moins lisible que les f-strings

### f-strings (recommandé)

```python
message = f"Bonjour {nom}, vous avez {age} ans."
```

**Avantages** :
- ✅ Syntaxe concise et lisible
- ✅ Évaluation d'expressions directement
- ✅ Plus rapide (évaluation au moment de l'exécution)
- ✅ Debugging intégré avec `{variable=}`

---

## 11) Cas d'usage pratiques

Voici quelques exemples concrets d'utilisation des f-strings dans des situations réelles.

### Générer un tableau

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

**Résultat** :

```
Produit           Prix   Qté    Total
----------------------------------------
Livre              12.50     2    25.00
Stylo               1.20     5     6.00
Cahier              3.00     4    12.00
```

### Générer du SQL dynamiquement

```python
table = "utilisateurs"
colonnes = ["nom", "email", "age"]
valeurs = ["Alice", "alice@example.com", 30]

query = f"""
INSERT INTO {table} ({', '.join(colonnes)})
VALUES ({', '.join(f"'{v}'" for v in valeurs)})
"""

print(query)
# INSERT INTO utilisateurs (nom, email, age)
# VALUES ('Alice', 'alice@example.com', '30')
```

**⚠️ Attention** : pour du vrai code SQL, utilisez des requêtes paramétrées pour éviter les injections SQL.

### Logs avec horodatage

```python
from datetime import datetime

def log(message):
    timestamp = datetime.now()
    print(f"[{timestamp:%Y-%m-%d %H:%M:%S}] {message}")

log("Application démarrée")
log("Connexion à la base de données réussie")
# [2026-02-09 14:30:45] Application démarrée
# [2026-02-09 14:30:46] Connexion à la base de données réussie
```

---

## 12) Pièges à éviter

Certaines erreurs courantes peuvent survenir avec les f-strings. Voici comment les éviter.

### 1. Variables non définies

```python
# Erreur : nom n'est pas défini
print(f"Bonjour {nom}")
# NameError: name 'nom' is not defined
```

### 2. F-strings et backslashes

Les backslashes ne sont **pas autorisés** directement dans les f-strings :

```python
# ❌ Erreur
print(f"Chemin : {os.path.join('C:', 'Users', 'Alice')}")

# ✅ Solution : stocker le résultat dans une variable
chemin = os.path.join('C:', 'Users', 'Alice')
print(f"Chemin : {chemin}")
```

### 3. Guillemets imbriqués

```python
# ❌ Erreur de syntaxe
print(f"Message : {"Hello"}")

# ✅ Solution : alterner les guillemets
print(f"Message : {'Hello'}")
print(f'Message : {"Hello"}')
```

---

## 13) Bonnes pratiques

### ✅ À faire

1. **Utiliser les f-strings par défaut** (Python 3.6+)
2. **Formater les nombres avec précision** : `{prix:.2f}`
3. **Utiliser `{variable=}` pour le debugging**
4. **Aligner les tableaux** avec `<`, `>`, `^`
5. **Extraire les expressions complexes** dans des variables

```python
# ❌ Difficile à lire
print(f"Total : {sum([p['prix'] * p['qte'] for p in produits]):.2f}€")

# ✅ Plus clair
total = sum(p['prix'] * p['qte'] for p in produits)
print(f"Total : {total:.2f}€")
```

### ❌ À éviter

- Utiliser `%` ou `.format()` sans raison (hors compatibilité Python < 3.6)
- Expressions trop complexes dans les f-strings
- F-strings dans des boucles intensives si la performance est critique (envisager `join`)

---

## Conclusion

Les f-strings sont la méthode moderne, performante et lisible pour formater des chaînes en Python. Elles remplacent avantageusement `%` et `.format()` dans la quasi-totalité des cas d'usage.

**Points clés à retenir :**

- Syntaxe : `f"Texte {variable}"`
- Évaluation d'expressions : `f"{x + y}"`
- Formatage de nombres : `{prix:.2f}`, `{nombre:,}`
- Alignement : `{texte:<10}`, `{texte:>10}`, `{texte:^10}`
- Debugging : `{variable=}` (Python 3.8+)

Adoptez les f-strings dès maintenant pour un code plus propre et plus lisible !

---

## Voir aussi

- [Comment faire des group by en Python]({% post_url 2025-10-08-Comment-faire-des-group-by-en-python %})
- [Comment créer une CLI en Python]({% post_url 2025-12-28-Comment-creer-une-CLI-en-python %})
- [Comment envoyer des emails en Python]({% post_url 2026-02-02-Comment-envoyer-des-emails-en-Python %})
- [Documentation officielle des f-strings](https://docs.python.org/3/reference/lexical_analysis.html#f-strings)
- [PEP 498 – Literal String Interpolation](https://peps.python.org/pep-0498/)
