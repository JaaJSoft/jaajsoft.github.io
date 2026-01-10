---
layout: article
title: "Sed : éditer des fichiers en ligne de commande avec des regex"
author: Pierre Chopinet
tags:
  - linux
  - sed
  - regex
  - cli
  - shell
  - bash
  - outils
---

`sed` (Stream EDitor) est un outil en ligne de commande pour éditer des flux de texte avec des expressions régulières. Indispensable pour les opérations de recherche/remplacement, suppression de lignes, insertion, et transformations sur fichiers ou flux stdin.
<!--more-->

Objectifs de l'article :
- Comprendre la syntaxe de base de `sed`
- Maîtriser les commandes essentielles (substitution, suppression, insertion, modification)
- Utiliser les expressions régulières avec `sed`
- Découvrir 15 cas pratiques du quotidien
- Connaître les options importantes (`-i`, `-n`, `-e`, `-r`/`-E`)

---

## Installation

`sed` est préinstallé sur la plupart des systèmes Unix/Linux. Sur macOS, la version BSD est installée par défaut (syntaxe légèrement différente).

### GNU sed (recommandé)

- **Linux** : déjà installé
- **macOS** : `brew install gnu-sed` (commande : `gsed`)
- **Windows** : WSL, Git Bash, ou Cygwin

Vérification :
```bash
sed --version
# GNU sed version 4.8 ou supérieur
```

---

## Syntaxe de base

```bash
sed 'commande' fichier.txt              # Affiche le résultat (ne modifie pas le fichier)
sed -i 'commande' fichier.txt           # Modifie le fichier en place
sed -i.bak 'commande' fichier.txt       # Modifie + crée une sauvegarde .bak
sed -e 'cmd1' -e 'cmd2' fichier.txt     # Plusieurs commandes
echo "texte" | sed 'commande'           # Depuis stdin
```

**Structure d'une commande sed** :
```
[adresse]commande[options]
```

- **adresse** : ligne(s) concernée(s) (optionnel, si absent = toutes les lignes)
- **commande** : `s` (substitute), `d` (delete), `p` (print), `a` (append), `i` (insert), `c` (change)...

---

## Les commandes essentielles

### 1) Substitution : `s/pattern/replacement/flags`

La commande la plus utilisée. Remplace `pattern` par `replacement`.

**Flags courants** :
- `g` : global (toutes les occurrences sur la ligne)
- `i` : insensible à la casse
- `2` : uniquement la 2ème occurrence
- `p` : afficher les lignes modifiées

```bash
# Remplacer la première occurrence
sed 's/foo/bar/' file.txt

# Remplacer toutes les occurrences
sed 's/foo/bar/g' file.txt

# Insensible à la casse
sed 's/foo/bar/gi' file.txt
```

### 2) Suppression : `d`

```bash
# Supprimer la ligne 3
sed '3d' file.txt

# Supprimer les lignes 2 à 5
sed '2,5d' file.txt

# Supprimer les lignes contenant "error"
sed '/error/d' file.txt

# Supprimer les lignes vides
sed '/^$/d' file.txt
```

### 3) Insertion et ajout : `i` et `a`

```bash
# Insérer avant la ligne 3
sed '3i\Nouvelle ligne' file.txt

# Ajouter après la ligne 3
sed '3a\Nouvelle ligne' file.txt

# Ajouter après chaque ligne contenant "pattern"
sed '/pattern/a\Ligne ajoutée' file.txt
```

### 4) Remplacement de ligne : `c`

```bash
# Remplacer la ligne 2
sed '2c\Nouveau contenu' file.txt

# Remplacer les lignes contenant "old"
sed '/old/c\Nouveau contenu' file.txt
```

### 5) Affichage sélectif : `p` avec `-n`

Par défaut, `sed` affiche toutes les lignes. `-n` supprime l'affichage automatique.

```bash
# Afficher uniquement la ligne 5
sed -n '5p' file.txt

# Afficher les lignes 10 à 20
sed -n '10,20p' file.txt

# Afficher les lignes contenant "error"
sed -n '/error/p' file.txt

# Équivalent de grep
sed -n '/pattern/p' file.txt  # = grep 'pattern' file.txt
```

---

## Expressions régulières avec sed

### Syntaxe par défaut (BRE - Basic Regular Expressions)

```bash
# Caractères spéciaux : . * ^ $ [ ] \
sed 's/^/#/' file.txt              # Commenter toutes les lignes
sed 's/ *$//' file.txt             # Supprimer espaces de fin de ligne
sed 's/[0-9]\+/NUM/g' file.txt     # Remplacer nombres par NUM (BRE: \+)
```

### Extended Regular Expressions (`-E` ou `-r`)

Option `-E` (GNU/BSD moderne) ou `-r` (GNU ancien) pour utiliser ERE.

```bash
# Avec ERE : + ? | () {} sans backslash
sed -E 's/[0-9]+/NUM/g' file.txt           # + sans backslash
sed -E 's/(foo|bar)/baz/g' file.txt        # alternance
sed -E 's/([0-9]{2})-([0-9]{2})/\2-\1/' file.txt  # groupes capturants
```

### Groupes capturants et backreferences

```bash
# Inverser deux mots séparés par un espace
sed -E 's/(\w+) (\w+)/\2 \1/' file.txt

# Doubler chaque mot
sed -E 's/\b(\w+)\b/\1 \1/g' file.txt

# Remplacer dates DD/MM/YYYY par YYYY-MM-DD
sed -E 's|([0-9]{2})/([0-9]{2})/([0-9]{4})|\3-\2-\1|g' file.txt
```

---

## 15 cas pratiques

### 1) Rechercher et remplacer (comme find & replace)

```bash
# Remplacer "http://" par "https://" dans tous les fichiers .txt
find . -name "*.txt" -exec sed -i 's|http://|https://|g' {} +
```

### 2) Commenter/décommenter des lignes

```bash
# Commenter les lignes contenant "debug"
sed -i '/debug/s/^/#/' config.txt

# Décommenter les lignes commençant par #debug
sed -i 's/^#\(debug\)/\1/' config.txt
```

### 3) Supprimer les lignes vides

```bash
sed -i '/^$/d' file.txt
```

### 4) Supprimer les espaces/tabs en début et fin de ligne

```bash
# Début de ligne
sed 's/^[ \t]*//' file.txt

# Fin de ligne
sed 's/[ \t]*$//' file.txt

# Les deux
sed 's/^[ \t]*//; s/[ \t]*$//' file.txt
```

### 5) Remplacer uniquement sur certaines lignes

```bash
# Remplacer "foo" par "bar" uniquement lignes 10 à 20
sed '10,20s/foo/bar/g' file.txt

# Remplacer uniquement dans les lignes contenant "section"
sed '/section/s/old/new/g' file.txt
```

### 6) Numéroter les lignes

```bash
sed = file.txt | sed 'N; s/\n/\t/'
# ou avec nl (plus simple)
nl file.txt
```

### 7) Afficher les lignes entre deux motifs

```bash
# Afficher lignes entre START et END (inclus)
sed -n '/START/,/END/p' file.txt
```

### 8) Supprimer les commentaires

```bash
# Supprimer les lignes commençant par #
sed '/^#/d' file.txt

# Supprimer les commentaires # en fin de ligne
sed 's/#.*$//' file.txt
```

### 9) Remplacer une ligne spécifique par le contenu d'un fichier

```bash
# Remplacer ligne 5 par le contenu de insert.txt
sed -i '5r insert.txt' file.txt
```

### 10) Extraire des informations avec regex

```bash
# Extraire les adresses email
sed -nE 's/.*([a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}).*/\1/p' file.txt

# Extraire les URLs
sed -nE 's|.*(https?://[^[:space:]]+).*|\1|p' file.txt
```

### 11) Convertir minuscules/majuscules

```bash
# Tout en majuscules
sed 's/.*/\U&/' file.txt

# Tout en minuscules
sed 's/.*/\L&/' file.txt

# Première lettre en majuscule (GNU sed)
sed 's/\b\(.\)/\U\1/g' file.txt
```

> Note : `\U`, `\L` sont des extensions GNU sed. Pour BSD/macOS, utilisez `tr` à la place.

### 12) Remplacer les retours à la ligne

```bash
# Joindre toutes les lignes en une seule
sed ':a;N;$!ba;s/\n/ /g' file.txt

# Plus simple avec tr
tr '\n' ' ' < file.txt
```

### 13) Doubler l'espacement (ligne vide après chaque ligne)

```bash
sed G file.txt
```

### 14) Supprimer les doublons de lignes consécutives

```bash
# Comme uniq, mais avec sed
sed '$!N; /^\(.*\)\n\1$/!P; D' file.txt

# Plus simple : sort + uniq
sort file.txt | uniq
```

### 15) Rechercher et remplacer avec délimiteur personnalisé

Utile pour les chemins avec `/`.

```bash
# Au lieu de s/\/old\/path/\/new\/path/g
sed 's|/old/path|/new/path|g' file.txt

# Ou avec #
sed 's#/old/path#/new/path#g' file.txt
```

---

## Options importantes

### `-i` : Éditer en place (in-place)

```bash
# Modifie le fichier directement
sed -i 's/foo/bar/g' file.txt

# Crée une sauvegarde
sed -i.bak 's/foo/bar/g' file.txt
```

> **macOS/BSD** : `-i` nécessite un argument (même vide) : `sed -i '' 's/foo/bar/g' file.txt`

### `-n` : Mode silencieux (suppress automatic printing)

```bash
# Afficher uniquement les lignes modifiées
sed -n 's/foo/bar/gp' file.txt
```

### `-e` : Commandes multiples

```bash
sed -e 's/foo/bar/g' -e 's/hello/world/g' file.txt

# Équivalent avec ;
sed 's/foo/bar/g; s/hello/world/g' file.txt
```

### `-E` ou `-r` : Extended regex

```bash
# GNU sed
sed -E 's/[0-9]+/NUM/g' file.txt

# Ancien GNU / BSD
sed -r 's/[0-9]+/NUM/g' file.txt  # GNU
sed -E 's/[0-9]+/NUM/g' file.txt  # BSD/macOS
```

### `-f` : Lire les commandes depuis un fichier

```bash
# script.sed
s/foo/bar/g
s/hello/world/g
/^#/d

# Exécution
sed -f script.sed file.txt
```

---

## Différences GNU sed vs BSD sed (macOS)

| Fonctionnalité       | GNU sed              | BSD sed (macOS)           |
|----------------------|----------------------|---------------------------|
| `-i` sans backup     | `-i`                 | `-i ''`                   |
| Extended regex       | `-E` ou `-r`         | `-E`                      |
| `\+` (BRE)           | Supporté             | Supporté                  |
| `\U`, `\L` (casse)   | Supporté             | Non supporté              |
| `-z` (NULL)          | Supporté             | Non supporté              |

**Conseil macOS** : installez GNU sed avec Homebrew (`brew install gnu-sed`) et utilisez `gsed`.

---

## Bonnes pratiques

### ✅ À faire

- **Testez d'abord sans `-i`** : vérifiez le résultat avant de modifier le fichier
- **Créez une sauvegarde** : `sed -i.bak` pour garder l'original
- **Utilisez `-E`** : plus lisible pour les regex complexes
- **Délimiteur alternatif** : `s|/path|new|g` pour les chemins
- **Échappez les caractères spéciaux** : `\.` `\*` `\$` etc.

### ❌ À éviter

- Modifier directement sans test (risque de perte de données)
- Regex trop complexes : préférez `awk`, `perl` ou un script Python
- `sed` pour du parsing HTML/XML/JSON : utilisez des outils dédiés

---

## Sed vs autres outils

| Outil  | Usage                                     |
|--------|-------------------------------------------|
| `sed`  | Substitutions simples, suppression lignes |
| `awk`  | Traitement colonne, calculs, conditions   |
| `grep` | Recherche de motifs                       |
| `tr`   | Remplacement caractères                   |
| `perl` | Regex complexes, Unicode                  |

**Règle générale** : `sed` pour les transformations ligne par ligne simples, `awk` pour les traitements tabulaires, `perl`/Python pour les cas complexes.

---

## Ressources

### Regex testers en ligne
- [regex101.com](https://regex101.com/) (mode PCRE/ECMAScript)
- [regexr.com](https://regexr.com/)

### Documentation
- `man sed` : manuel local
- [GNU sed manual](https://www.gnu.org/software/sed/manual/)
- [sed one-liners](http://www.pement.org/sed/sed1line.txt) : collection de commandes utiles

---

## Conclusion

`sed` est un outil puissant pour éditer des fichiers texte en ligne de commande. Combiné avec les expressions régulières, il permet d'automatiser des tâches répétitives de transformation de texte.

**Points clés à retenir :**
- `s/pattern/replacement/g` : substitution
- `-i` : éditer en place (avec sauvegarde `-i.bak`)
- `-n` + `p` : affichage sélectif
- `-E` : regex étendues (plus lisibles)
- Testez toujours sans `-i` avant de modifier

Pour des opérations plus complexes (parsing structuré, calculs, conditions multiples), préférez `awk`, `jq` (JSON), ou un langage de script.

---

## Voir aussi

- [Comment manipuler du JSON en ligne de commande avec jq]({% post_url 2025-09-17-Comment-utiliser-jq %})
- [Linux : programmer une tâche avec cron]({% post_url 2025-10-11-Linux-programmer-une-tache-avec-cron %})
- [Comment transformer un JSON en CSV avec jq]({% post_url 2025-10-19-Comment-transformer-un-JSON-en-CSV-avec-jq %})
- [Comment changer le hostname en ligne de commande sur Ubuntu ou Debian]({% post_url 2025-09-13-Comment-changer-le-hostname-en-ligne-de-commande-sur-Ubuntu-ou-Debian %})
