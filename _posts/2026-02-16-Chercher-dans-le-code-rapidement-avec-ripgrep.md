---
layout: article
title: "Ripgrep (rg) : chercher dans le code à la vitesse de l'éclair"
author: Pierre Chopinet
tags:
  - linux
  - ripgrep
  - rg
  - grep
  - cli
  - shell
  - bash
  - outils
  - search
---

`ripgrep` (commande `rg`) est un outil de recherche en ligne de commande ultra-rapide, conçu pour chercher dans du code. Il est 10× plus rapide que `grep`, ignore automatiquement `.git`, `node_modules` et autres fichiers inutiles, et supporte nativement les regex Rust performantes.
<!--more-->

Objectifs de l'article :
- Installer `ripgrep` sur Linux/macOS/Windows
- Comprendre les différences avec `grep`
- Maîtriser les options essentielles (types de fichiers, contexte, remplacement)
- Découvrir 15 cas pratiques du quotidien
- Intégrer `rg` dans vos workflows de développement

---

## Pourquoi ripgrep ?

### Comparaison avec grep

| Critère                  | grep                   | ripgrep (rg)         |
|--------------------------|------------------------|----------------------|
| Vitesse                  | Rapide                 | **10× plus rapide**  |
| Ignore .git/node_modules | Non (besoin --exclude) | **Oui (par défaut)** |
| Regex                    | BRE/ERE                | **Rust (PCRE-like)** |
| Coloration syntaxe       | Basique                | **Excellente**       |
| Recherche récursive      | `-r` requis            | **Par défaut**       |
| Types de fichiers        | Manuel                 | **Auto-détection**   |

### Benchmark typique

```bash
# Rechercher "TODO" dans un projet React (50k fichiers)
time grep -r "TODO" .        # ~8 secondes
time rg "TODO"               # ~0.3 secondes (26× plus rapide)
```

Pourquoi cette différence ?
- `rg` ignore automatiquement `.git`, `node_modules`, fichiers binaires
- Multithreading natif (utilise tous les cœurs CPU)
- Optimisations mémoire et regex performantes

---

## Installation

### Linux

```bash
# Debian/Ubuntu
sudo apt install ripgrep

# Fedora
sudo dnf install ripgrep

# Arch
sudo pacman -S ripgrep

# Depuis les binaires (toutes distros basée sur debian)
curl -LO https://github.com/BurntSushi/ripgrep/releases/download/14.1.0/ripgrep_14.1.0-1_amd64.deb
sudo dpkg -i ripgrep_14.1.0-1_amd64.deb
```

### macOS

```bash
brew install ripgrep
```

### Windows

```bash
# Scoop
scoop install ripgrep

# Chocolatey
choco install ripgrep

# Cargo (si Rust installé)
cargo install ripgrep
```

Vérification :
```bash
rg --version
# ripgrep 14.1.0
```

---

## Syntaxe de base

```bash
rg pattern                          # Recherche récursive depuis le répertoire courant
rg pattern path/                    # Recherche dans un répertoire spécifique
rg pattern file.txt                 # Recherche dans un fichier
rg -i pattern                       # Insensible à la casse (case-insensitive)
rg -w pattern                       # Mots entiers uniquement (word boundary)
rg -v pattern                       # Inverser (lignes NE contenant PAS pattern)
```

**Différences clés avec grep :**
- `rg` est récursif **par défaut** (pas besoin de `-r`)
- `rg` ignore automatiquement `.gitignore`, fichiers binaires, `.git`, `node_modules`
- Les couleurs sont activées par défaut

---

## Les bases de ripgrep

### 1) Recherche simple

```bash
# Rechercher "TODO" dans tous les fichiers
rg TODO

# Rechercher "function" en ignorant la casse
rg -i function

# Rechercher le mot exact "test" (pas "testing", "contest")
rg -w test
```

Sortie typique :
```
src/utils.py
42:    # TODO: refactor this function
68:    # TODO: add error handling

src/api/routes.py
12:    # TODO: implement authentication
```

### 2) Limiter par type de fichier

```bash
# Uniquement dans les fichiers Python
rg TODO -t py

# Uniquement JavaScript/TypeScript
rg TODO -t js -t ts

# Exclure les fichiers Markdown
rg TODO -T md

# Lister les types disponibles
rg --type-list
```

Types courants : `py`, `js`, `ts`, `java`, `go`, `rust`, `c`, `cpp`, `md`, `html`, `css`, `json`, `yaml`

### 3) Recherche avec regex

```bash
# Regex basique : adresses email
rg '\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b'

# Numéros de téléphone français
rg '0[1-9]( [0-9]{2}){4}'

# Imports Python
rg '^import \w+|^from \w+ import'

# Fonctions JavaScript/TypeScript
rg 'function \w+\(|const \w+ = \('
```

### 4) Contexte (lignes avant/après)

```bash
# 3 lignes avant et après
rg -C 3 TODO

# 2 lignes avant
rg -B 2 TODO

# 5 lignes après
rg -A 5 TODO
```

Sortie avec contexte :
```
src/utils.py
39-def process_data(data):
40-    """Process incoming data."""
41-    validate(data)
42:    # TODO: refactor this function
43-    result = transform(data)
44-    return result
45-
```

---

## 15 cas pratiques

### 1) Trouver tous les TODOs/FIXMEs

```bash
# TODOs
rg 'TODO|FIXME|HACK|XXX'

# Avec contexte et fichiers Python uniquement
rg -C 2 'TODO|FIXME' -t py
```

### 2) Chercher une fonction/classe dans tout le projet

```bash
# Définition de fonction Python
rg 'def calculate_total'

# Classe Java
rg 'class UserService'

# Interface TypeScript
rg 'interface User \{'
```

### 3) Trouver des imports/includes

```bash
# Imports Python
rg '^from .* import|^import '

# Require Node.js
rg "require\(['\"]"

# Imports Java
rg '^import '
```

### 4) Chercher des valeurs de configuration

```bash
# Variables d'environnement
rg 'process\.env\.|os\.getenv'

# Clés API (pattern simple)
rg 'api_key|API_KEY|apiKey'

# URLs/endpoints
rg 'https?://[^\s"]+'
```

### 5) Détecter des mots de passe en dur (code smell)

```bash
# Patterns suspects
rg -i 'password\s*=\s*["\'][^"\']{3,}'

# Secrets potentiels
rg 'secret|token|key' -t py -t js
```

### 6) Compter les occurrences

```bash
# Nombre de lignes contenant "error"
rg error -c

# Total d'occurrences (pas de lignes)
rg error --count-matches
```

Sortie :
```
src/logger.py:4
src/api.py:12
src/utils.py:2
```

### 7) Lister uniquement les fichiers contenant un pattern

```bash
# Fichiers avec "deprecated"
rg -l deprecated

# Fichiers SANS "test"
rg -L test
```

### 8) Rechercher dans des fichiers spécifiques par extension

```bash
# Fichiers .env seulement
rg DATABASE_URL -g '*.env'

# Tous les fichiers sauf .min.js
rg pattern -g '!*.min.js'

# Fichiers de config (yaml, json, toml)
rg pattern -g '*.{yaml,json,toml}'
```

### 9) Remplacer du texte (prévisualisation)

```bash
# Voir à quoi ressemblerait le remplacement
rg 'old_function' -r 'new_function'

# Avec sed pour appliquer
rg -l 'old_function' | xargs sed -i 's/old_function/new_function/g'
```

> **Note** : `rg` ne modifie pas les fichiers directement. Utilisez `sed` ou un éditeur pour appliquer les changements.

### 10) Chercher des lignes longues (code smell)

```bash
# Lignes de plus de 120 caractères
rg '.{121,}'

# Uniquement Python
rg '.{121,}' -t py
```

### 11) Trouver des fichiers binaires/non-texte

```bash
# Forcer la recherche dans tous les fichiers (y compris binaires)
rg pattern --binary

# Lister les fichiers binaires détectés
rg --files --type-not text
```

### 12) Exclure des répertoires

```bash
# Exclure node_modules et vendor (normalement déjà fait)
rg pattern -g '!node_modules' -g '!vendor'

# Exclure tous les répertoires de test
rg pattern -g '!**/test/**' -g '!**/tests/**'
```

### 13) Recherche multi-ligne

```bash
# Activer le mode multiline
rg -U 'function.*\{.*\}' -t js

# Exemple : trouver des blocs try-catch Python
rg -U 'try:.*except.*:' -t py
```

### 14) Statistiques sur le code

```bash
# Nombre de fichiers Python
rg --files -t py | wc -l

# Nombre de lignes de code Python (approximatif)
rg --files -t py | xargs wc -l | tail -1

# Nombre de fonctions Python
rg '^def \w+' -t py -c | awk -F: '{sum+=$2} END {print sum}'
```

### 15) Recherche avec gitignore personnalisé

```bash
# Respecter .gitignore (par défaut)
rg pattern

# Ignorer .gitignore (chercher partout)
rg pattern --no-ignore

# Utiliser un fichier ignore custom
rg pattern --ignore-file .rgignore
```

Exemple `.rgignore` :
```
# .rgignore
*.log
*.tmp
build/
dist/
```

---

## Options essentielles

### Comportement de recherche

- `-i`, `--ignore-case` : insensible à la casse
- `-w`, `--word-regexp` : mots entiers uniquement
- `-v`, `--invert-match` : lignes NE contenant PAS le pattern
- `-U`, `--multiline` : recherche multi-lignes
- `-F`, `--fixed-strings` : recherche littérale (pas de regex)

### Contexte

- `-C NUM`, `--context NUM` : NUM lignes avant et après
- `-B NUM`, `--before-context NUM` : NUM lignes avant
- `-A NUM`, `--after-context NUM` : NUM lignes après

### Filtrage de fichiers

- `-t TYPE`, `--type TYPE` : type de fichier (py, js, etc.)
- `-T TYPE`, `--type-not TYPE` : exclure un type
- `-g GLOB`, `--glob GLOB` : pattern de fichier (ex: `*.py`)
- `-g '!GLOB'` : exclure un pattern

### Sortie

- `-l`, `--files-with-matches` : uniquement noms de fichiers
- `-L`, `--files-without-match` : fichiers sans correspondance
- `-c`, `--count` : nombre de lignes par fichier
- `--count-matches` : nombre total d'occurrences
- `-o`, `--only-matching` : uniquement la partie qui matche
- `--no-line-number` : masquer les numéros de ligne
- `--no-filename` : masquer les noms de fichiers

### Performance et comportement

- `--no-ignore` : ne pas respecter .gitignore
- `--hidden` : inclure les fichiers cachés (`.dotfiles`)
- `--no-binary` : ignorer les fichiers binaires (défaut)
- `-j NUM`, `--threads NUM` : nombre de threads (auto par défaut)

---

## Configuration personnalisée

### Fichier de config

Créez `~/.config/ripgrep/config` (ou `~/.ripgreprc`) :

```bash
# ~/.ripgreprc
# Toujours afficher le contexte
--context=2

# Ignorer les fichiers minifiés
--glob=!*.min.js
--glob=!*.min.css

# Smart case : insensible si lowercase, sensible si uppercase
--smart-case

# Afficher les fichiers cachés
--hidden

# Nombre max de threads
--threads=8
```

Utilisation :
```bash
# Utilise automatiquement ~/.ripgreprc
rg pattern

# Ignorer le fichier de config
rg pattern --no-config
```

### Types de fichiers personnalisés

```bash
# Ajouter un type "web" (html, css, js)
rg pattern --type-add 'web:*.{html,css,js}' -t web

# Type pour fichiers de config
rg pattern --type-add 'config:*.{yaml,yml,json,toml,ini}' -t config
```

Ajoutez dans `~/.ripgreprc` pour les rendre permanents :
```bash
--type-add=web:*.{html,css,js}
--type-add=config:*.{yaml,yml,json,toml,ini}
```

---

## Intégration avec d'autres outils

### Avec fzf (fuzzy finder)

```bash
# Recherche interactive de fichiers
rg --files | fzf

# Recherche interactive dans le contenu
rg --line-number --no-heading --color=always pattern | fzf --ansi
```

### Avec vim/neovim

```vim
" .vimrc / init.vim
" Utiliser rg au lieu de grep
set grepprg=rg\ --vimgrep\ --smart-case\ --follow

" Raccourci pour chercher le mot sous le curseur
nnoremap <leader>g :grep <C-R><C-W><CR>
```

### Avec VS Code

Installez l'extension "ripgrep" ou configurez :
```json
{
  "search.useRipgrep": true,
  "search.ripgrep.args": [
    "--hidden",
    "--glob=!.git"
  ]
}
```

### Avec git (chercher dans l'historique)

```bash
# Chercher dans les commits
git log -S "old_function" --source --all

# Chercher dans les fichiers non commités
rg pattern $(git diff --name-only)
```

---

## Comparaison ripgrep vs autres outils

| Outil                  | Usage                         | Quand l'utiliser ?              |
|------------------------|-------------------------------|---------------------------------|
| `rg`                   | Recherche rapide dans le code | **Toujours pour du code**       |
| `grep`                 | Recherche basique             | Scripts POSIX, systèmes sans rg |
| `ag` (silver searcher) | Recherche dans le code        | Alternative à rg (plus lent)    |
| `ack`                  | Recherche Perl-like           | Legacy, préférer rg             |
| `find + grep`          | Recherche avec contrôle fin   | Cas très spécifiques            |

**Règle générale** : utilisez `rg` pour tout ce qui touche au code. C'est plus rapide, plus simple, et mieux intégré avec les outils modernes.

---

## Bonnes pratiques

### ✅ À faire

- **Utilisez `-t` pour limiter les types de fichiers** : plus rapide et pertinent
- **Créez un `.rgignore`** pour exclure des patterns spécifiques à votre projet
- **Combinez avec `fzf`** pour des recherches interactives
- **Utilisez `--hidden`** si vous cherchez dans des dotfiles
- **Activez smart-case** (`--smart-case`) : insensible si lowercase, sensible sinon

### ❌ À éviter

- Ne pas utiliser `rg` pour modifier des fichiers (utilisez `sed` après avoir trouvé)
- Éviter `--no-ignore` sans raison : performances dégradées
- Ne pas utiliser `rg` pour parser du JSON/XML (utilisez `jq`, `xmllint`)

---

## Cheatsheet

### Commandes courantes
```bash
rg pattern                    # Recherche récursive
rg -i pattern                 # Case-insensitive
rg -w pattern                 # Mots entiers
rg -v pattern                 # Inverser (NOT)
rg -C 3 pattern               # 3 lignes de contexte
rg -t py pattern              # Fichiers Python uniquement
rg -g '*.js' pattern          # Glob pattern
rg -l pattern                 # Noms de fichiers seulement
rg -c pattern                 # Comptage par fichier
```

---

## Conclusion

`ripgrep` est l'outil de recherche de code moderne par excellence. Sa vitesse, son intelligence (respect de .gitignore), et sa simplicité en font un remplacement naturel de `grep` pour tous les développeurs.

**Points clés à retenir :**
- `rg pattern` : recherche récursive intelligente
- `-t TYPE` : filtrer par type de fichier
- `-C NUM` : contexte avant/après
- `--no-ignore` : forcer la recherche partout
- Configuration via `~/.ripgreprc`

Pour des opérations complémentaires, pensez à combiner `rg` avec `sed`, `awk`, `fzf`, ou votre éditeur favori.

---

## Voir aussi

- [Sed : éditer des fichiers en ligne de commande avec des regex]({% post_url 2026-01-19-Sed-editer-des-fichiers-en-ligne-de-commande %})
- [Comment manipuler du JSON en ligne de commande avec jq]({% post_url 2025-09-17-Comment-utiliser-jq %})
- [Linux : programmer une tâche avec cron]({% post_url 2025-10-11-Linux-programmer-une-tache-avec-cron %})
- [Installer et configurer Fail2ban sur Ubuntu/Debian]({% post_url 2025-09-21-Installer-et-configurer-Fail2ban-sur-Ubuntu-Debian %})
