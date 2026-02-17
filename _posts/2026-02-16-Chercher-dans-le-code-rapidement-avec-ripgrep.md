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
- Maîtriser les options essentielles (types de fichiers, contexte, smart-case, regex avancées)
- Découvrir 15 cas pratiques du quotidien
- Intégrer `rg` dans vos workflows (fzf, vim, VS Code, scripts)

---

## Pourquoi ripgrep ?

### Comparaison avec grep

| Critère                  | grep                   | ripgrep (rg)           |
|--------------------------|------------------------|------------------------|
| Vitesse                  | Rapide                 | **10-30× plus rapide** |
| Ignore .git/node_modules | Non (besoin --exclude) | **Oui (par défaut)**   |
| Regex                    | BRE/ERE                | **Rust (PCRE2 en option)** |
| Coloration syntaxe       | Basique                | **Excellente**         |
| Recherche récursive      | `-r` requis            | **Par défaut**         |
| Types de fichiers        | Manuel                 | **Auto-détection**     |
| Smart case               | Non                    | **Oui (`-S`)**         |

### D'où vient la différence de vitesse ?

- **Ignore intelligent** : `rg` respecte `.gitignore` et ignore `.git`, `node_modules`, fichiers binaires — sans configuration
- **Multithreading natif** : utilise tous les cœurs CPU disponibles
- **Moteur regex optimisé** : le moteur Rust est compilé en automate fini, évitant le backtracking catastrophique

L'écart dépend du projet : sur un dépôt propre avec peu de fichiers ignorés, `rg` est ~10× plus rapide. Sur un projet avec un gros `node_modules` ou beaucoup de binaires, le gain peut dépasser 30×.

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
```

Pour installer la dernière version manuellement, récupérez le `.deb` depuis la [page des releases GitHub](https://github.com/BurntSushi/ripgrep/releases) :

```bash
# Exemple pour la version 14.1.1
curl -LO https://github.com/BurntSushi/ripgrep/releases/download/14.1.1/ripgrep_14.1.1-1_amd64.deb
sudo dpkg -i ripgrep_14.1.1-1_amd64.deb
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

# Winget
winget install BurntSushi.ripgrep.MSVC

# Cargo (si Rust installé)
cargo install ripgrep
```

Vérification :
```bash
rg --version
```

---

## Les essentiels

### Recherche de base

```bash
rg pattern                          # Recherche récursive depuis le répertoire courant
rg pattern path/                    # Recherche dans un répertoire spécifique
rg pattern fichier.txt              # Recherche dans un fichier
```

Sortie typique :
```
src/utils.py
42:    # TODO: refactor this function
68:    # TODO: add error handling

src/api/routes.py
12:    # TODO: implement authentication
```

**Différences clés avec grep :**
- `rg` est récursif **par défaut** (pas besoin de `-r`)
- `rg` ignore automatiquement `.gitignore`, fichiers binaires, `.git`, `node_modules`
- Les couleurs sont activées par défaut

### Casse et mots

```bash
rg -i pattern                       # Insensible à la casse
rg -S pattern                       # Smart case : insensible si pattern en minuscules,
                                     #              sensible dès qu'il y a une majuscule
rg -w test                          # Mots entiers ("test" mais pas "testing" ni "contest")
rg -v pattern                       # Inverser : lignes NE contenant PAS pattern
rg -F 'foo(bar)'                    # Recherche littérale (pas de regex)
```

`-S` (smart case) est probablement l'option la plus pratique au quotidien. `rg -S todo` trouvera "TODO", "Todo", "todo", mais `rg -S TODO` ne trouvera que "TODO".

### Filtrer par type de fichier

```bash
# Uniquement les fichiers Python
rg TODO -t py

# JavaScript et TypeScript
rg TODO -t js -t ts

# Exclure les fichiers Markdown
rg TODO -T md

# Lister les types disponibles
rg --type-list
```

Types courants : `py`, `js`, `ts`, `java`, `go`, `rust`, `c`, `cpp`, `md`, `html`, `css`, `json`, `yaml`

### Contexte (lignes avant/après)

```bash
rg -C 3 TODO                        # 3 lignes avant et après
rg -B 2 TODO                        # 2 lignes avant
rg -A 5 TODO                        # 5 lignes après
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

### Contrôle de la sortie

```bash
rg -l pattern                       # Uniquement les noms de fichiers
rg -L pattern                       # Fichiers SANS correspondance
rg -c pattern                       # Nombre de lignes matchant par fichier
rg --count-matches pattern          # Nombre total d'occurrences par fichier
rg -o pattern                       # Uniquement la partie qui matche
rg --no-line-number pattern         # Masquer les numéros de ligne
rg --no-filename pattern            # Masquer les noms de fichiers
```

---

## 15 cas pratiques

### 1) Trouver tous les TODOs et FIXMEs d'un projet

```bash
rg 'TODO|FIXME|HACK|XXX' -C 1
```

Ajoutez `-t py` ou `-t js` pour cibler un langage spécifique.

### 2) Chercher une définition de fonction ou de classe

```bash
# Définition Python
rg 'def calculate_total'

# Classe Java
rg 'class UserService'

# Interface TypeScript
rg 'interface UserProps \{'
```

### 3) Détecter des secrets en dur dans le code

```bash
# Mots de passe en dur
rg -i 'password\s*=\s*["\047][^"\047]{3,}'

# Clés API / tokens suspects
rg -S 'api_key|apiKey|API_KEY|secret_key|access_token' -t py -t js -t ts
```

Utile en revue de code ou avant un commit, pour éviter de pousser des credentials.

### 4) Chercher dans des logs ou la sortie d'une commande

```bash
# Filtrer les erreurs dans un fichier de log
rg 'ERROR|WARN' /var/log/app.log

# Piping depuis une autre commande
docker logs mon-container 2>&1 | rg 'timeout|connection refused'

# Filtrer la sortie de kubectl
kubectl get pods -A | rg 'CrashLoop|Error'
```

`rg` lit stdin quand il ne reçoit pas de fichier en argument, ce qui le rend utilisable partout où vous utiliseriez `grep` dans un pipe.

### 5) Chercher des imports et dépendances

```bash
# Qui importe ce module Python ?
rg '^from utils import|^import utils'

# Require Node.js
rg "require\(['\"]express"

# Imports Go
rg '"github\.com/.*"' -t go
```

### 6) Compter les occurrences dans un projet

```bash
# Nombre de lignes contenant "error" par fichier
rg error -c

# Total d'occurrences (pas de lignes) par fichier
rg error --count-matches

# Compter dans tout le projet
rg error --count-matches | awk -F: '{sum+=$NF} END {print sum}'
```

### 7) Filtrer par glob / exclure des fichiers

```bash
# Uniquement les fichiers .env
rg DATABASE_URL -g '*.env'

# Tous les fichiers sauf les minifiés
rg pattern -g '!*.min.js' -g '!*.min.css'

# Fichiers de config uniquement
rg pattern -g '*.{yaml,yml,json,toml}'
```

### 8) Prévisualiser un remplacement

```bash
# Voir le résultat d'un remplacement (sans modifier les fichiers)
rg 'old_function' -r 'new_function'

# Avec capture groups
rg 'log\((\w+)\)' -r 'logger.info($1)'

# Appliquer avec sed une fois satisfait
rg -l 'old_function' | xargs sed -i 's/old_function/new_function/g'
```

> `rg -r` ne modifie jamais les fichiers, il affiche juste la sortie transformée. Utilisez `sed` ou votre éditeur pour appliquer.

### 9) Recherche multi-ligne

```bash
# Activer le mode multiline avec -U
rg -U 'try:.*?except' -t py

# Trouver des fonctions vides en JavaScript
rg -U 'function \w+\([^)]*\)\s*\{\s*\}' -t js
```

Le flag `-U` permet au `.` de matcher les retours à la ligne.

### 10) Regex avancées avec PCRE2

```bash
# Activer le moteur PCRE2 (lookahead, lookbehind, backreferences)
rg -P '(?<=def )\w+(?=\()' -t py        # Noms de fonctions Python (lookbehind/lookahead)

# Lignes contenant "error" mais PAS "404"
rg -P 'error(?!.*404)'

# Backreference : mots doublés
rg -P '\b(\w+)\s+\1\b'
```

> `-P` nécessite que `ripgrep` soit compilé avec le support PCRE2 (c'est le cas sur la plupart des distributions).

### 11) Exclure des répertoires

```bash
# Exclure les dossiers de test
rg pattern -g '!**/test/**' -g '!**/tests/**' -g '!**/__tests__/**'

# Exclure vendor et build
rg pattern -g '!vendor' -g '!build' -g '!dist'
```

Note : `node_modules` et `.git` sont déjà exclus par défaut via `.gitignore`.

### 12) Chercher dans les fichiers cachés et ignorés

```bash
# Inclure les fichiers cachés (.dotfiles)
rg pattern --hidden

# Ignorer .gitignore (chercher partout, y compris node_modules)
rg pattern --no-ignore

# Les deux combinés
rg pattern --hidden --no-ignore
```

Utile pour chercher dans `.env`, `.github/`, ou d'autres fichiers cachés.

### 13) Sortie JSON pour les scripts

```bash
# Sortie JSON structurée
rg TODO --json

# Exploitable avec jq
rg TODO --json | jq 'select(.type == "match") | .data.path.text'
```

La sortie `--json` donne un objet par ligne, avec le chemin, le numéro de ligne, et le contenu. Idéal pour intégrer `rg` dans des pipelines de CI ou des scripts d'analyse.

### 14) Statistiques rapides sur le code

```bash
# Nombre de fichiers Python dans le projet
rg --files -t py | wc -l

# Nombre de fonctions Python
rg '^def \w+' -t py -c | awk -F: '{sum+=$NF} END {print sum}'

# Nombre de classes Java
rg 'class \w+' -t java -c | awk -F: '{sum+=$NF} END {print sum}'
```

### 15) Utiliser un fichier d'exclusion personnalisé

```bash
# Respecter .gitignore (par défaut)
rg pattern

# Utiliser un fichier d'exclusion dédié
rg pattern --ignore-file .rgignore

# Ignorer le fichier de config ripgrep
rg pattern --no-config
```

Exemple de `.rgignore` à placer à la racine de votre projet :
```
*.log
*.tmp
*.min.js
*.min.css
build/
dist/
coverage/
```

---

## Options essentielles (référence rapide)

### Comportement de recherche

| Option | Description |
|--------|-------------|
| `-i`, `--ignore-case` | Insensible à la casse |
| `-S`, `--smart-case` | Insensible si minuscules, sensible si majuscule dans le pattern |
| `-w`, `--word-regexp` | Mots entiers uniquement |
| `-v`, `--invert-match` | Lignes NE contenant PAS le pattern |
| `-U`, `--multiline` | Recherche multi-lignes |
| `-P`, `--pcre2` | Moteur PCRE2 (lookahead, lookbehind) |
| `-F`, `--fixed-strings` | Recherche littérale (pas de regex) |

### Contexte et sortie

| Option | Description |
|--------|-------------|
| `-C NUM` | NUM lignes avant et après |
| `-B NUM` | NUM lignes avant |
| `-A NUM` | NUM lignes après |
| `-l` | Uniquement noms de fichiers |
| `-c` | Nombre de lignes par fichier |
| `--count-matches` | Nombre d'occurrences par fichier |
| `-o` | Uniquement la partie qui matche |
| `-r TEXTE` | Prévisualiser un remplacement |
| `--json` | Sortie JSON structurée |

### Filtrage de fichiers

| Option | Description |
|--------|-------------|
| `-t TYPE` | Type de fichier (py, js, etc.) |
| `-T TYPE` | Exclure un type |
| `-g GLOB` | Pattern de fichier (ex: `*.py`) |
| `-g '!GLOB'` | Exclure un pattern |
| `--hidden` | Inclure les fichiers cachés |
| `--no-ignore` | Ignorer .gitignore |
| `--ignore-file` | Fichier d'exclusion personnalisé |

### Performance

| Option | Description |
|--------|-------------|
| `-j NUM` | Nombre de threads (auto par défaut) |
| `--no-binary` | Ignorer les fichiers binaires (défaut) |

---

## Configuration personnalisée

### Fichier de config

Créez `~/.config/ripgrep/config` (ou `~/.ripgreprc` avec `export RIPGREP_CONFIG_PATH=~/.ripgreprc`) :

```bash
# ~/.ripgreprc

# Smart case par défaut
--smart-case

# Toujours afficher 2 lignes de contexte
--context=2

# Ignorer les fichiers minifiés
--glob=!*.min.js
--glob=!*.min.css

# Inclure les fichiers cachés
--hidden

# Mais toujours exclure .git
--glob=!.git
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

### Aliases utiles

Ajoutez à votre `~/.bashrc` ou `~/.zshrc` :

```bash
# Recherche dans le code uniquement (ignore tests)
alias rgs='rg -g "!**/test/**" -g "!**/tests/**" -g "!**/__tests__/**"'

# Recherche TODO/FIXME rapide
alias todos='rg "TODO|FIXME|HACK|XXX" -C 1'

# Recherche avec fzf interactif
rgf() {
  rg --line-number --no-heading --color=always "$@" | fzf --ansi --preview 'echo {}'
}
```

---

## Intégration avec d'autres outils

### Avec fzf (fuzzy finder)

```bash
# Recherche interactive de fichiers
rg --files | fzf

# Recherche interactive dans le contenu avec prévisualisation
rg --line-number --no-heading --color=always "" | fzf --ansi \
  --preview 'bat --color=always {1} --highlight-line {2}' \
  --delimiter ':'
```

### Avec vim/neovim

```vim
" .vimrc / init.vim
" Utiliser rg comme grepprg
set grepprg=rg\ --vimgrep\ --smart-case
set grepformat=%f:%l:%c:%m

" Raccourci pour chercher le mot sous le curseur
nnoremap <leader>g :grep <C-R><C-W><CR>
```

### Avec VS Code

VS Code utilise déjà ripgrep en interne pour sa recherche. Pour personnaliser :
```json
{
  "search.useRipgrep": true,
  "search.ripgrep.args": [
    "--hidden",
    "--glob=!.git"
  ]
}
```

### Avec git

```bash
# Chercher un pattern dans les fichiers modifiés (non commités)
rg pattern $(git diff --name-only)

# Chercher un pattern dans les fichiers d'un commit
git show --name-only HEAD | tail -n +7 | xargs rg pattern

# Pour chercher dans l'historique git (contenu supprimé), utilisez git log :
git log -S "old_function" --source --all --oneline
```

---

## Comparaison avec les alternatives

| Outil                  | Usage                         | Verdict                         |
|------------------------|-------------------------------|---------------------------------|
| `rg`                   | Recherche rapide dans le code | **Le meilleur choix général**   |
| `grep`                 | Recherche basique             | Scripts POSIX, systèmes sans rg |
| `ag` (silver searcher) | Recherche dans le code        | Correct mais plus lent que rg   |
| `ack`                  | Recherche Perl-like           | Legacy, préférer rg             |

---

## Conclusion

`ripgrep` remplace avantageusement `grep` pour toute recherche dans du code. Ses points forts : la vitesse, le respect automatique de `.gitignore`, et le smart case. Une fois installé, il n'y a quasiment aucune raison de revenir à `grep` pour chercher dans un projet.

**Les options à retenir en priorité :**
- `rg pattern` : recherche récursive intelligente
- `-S` : smart case (à mettre dans votre config)
- `-t TYPE` : filtrer par type de fichier
- `-C NUM` : contexte avant/après
- `-P` : regex avancées (lookahead, lookbehind)
- `--json` : sortie structurée pour les scripts

---

## Voir aussi

- [Sed : éditer des fichiers en ligne de commande avec des regex]({% post_url 2026-01-19-Sed-editer-des-fichiers-en-ligne-de-commande %})
- [Comment manipuler du JSON en ligne de commande avec jq]({% post_url 2025-09-17-Comment-utiliser-jq %})
- [Comment transformer un JSON en CSV avec jq]({% post_url 2025-10-19-Comment-transformer-un-JSON-en-CSV-avec-jq %})
- [Comment créer une CLI en Python]({% post_url 2025-12-28-Comment-creer-une-CLI-en-python %})
