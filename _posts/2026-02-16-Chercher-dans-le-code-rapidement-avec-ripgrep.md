---
layout: article
title: "Ripgrep (rg) : chercher rapidement dans le code"
description: "Chercher dans le code avec ripgrep (rg) : fichiers ignorés par défaut, filtres par type et par glob, regex PCRE2, configuration, fzf, Vim et git."
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

Pour retrouver une fonction, un TODO ou une variable de configuration dans un projet, on tape souvent `grep -r`, qui fouille aussi `.git`, `node_modules` et tous les fichiers générés. ripgrep (commande `rg`) les ignore par défaut : les résultats sont plus pertinents, et ils arrivent beaucoup plus vite sur un vrai projet. Dans ce tutoriel, nous allons voir ses options les plus utiles sur un petit projet d'exemple.
<!--more-->

Dans cet article :
- Installation
- La recherche de base
- Ce que rg ignore par défaut
- Casse, mots entiers et texte littéral
- Filtrer par type de fichier ou par glob
- Contexte et format de la sortie
- Prévisualiser un remplacement
- Chercher sur plusieurs lignes
- Les regex PCRE2
- Le fichier de configuration
- Utiliser rg avec fzf, Vim, VS Code et git

## Installation

ripgrep est disponible dans les dépôts des principales distributions et dans les gestionnaires de paquets courants :

```bash
sudo apt install ripgrep                  # Debian, Ubuntu
sudo dnf install ripgrep                  # Fedora
sudo pacman -S ripgrep                    # Arch
brew install ripgrep                      # macOS
winget install BurntSushi.ripgrep.MSVC    # Windows (ou scoop install ripgrep, ou choco install ripgrep)
cargo install ripgrep                     # Depuis les sources, avec Rust
```

Le paquet s'appelle `ripgrep`, mais la commande est `rg`. La version des dépôts est parfois un peu ancienne : la [page des releases](https://github.com/BurntSushi/ripgrep/releases) du projet propose des binaires à jour, dont un paquet `.deb`.

Les exemples de cet article ont été testés avec ripgrep 14.1.0, la version du paquet d'Ubuntu 24.04. Pour connaître la vôtre : `rg --version`.

## La recherche de base

Les exemples portent sur un petit projet Python et JavaScript versionné avec git. Son `.gitignore` exclut le dossier `node_modules/`, les fichiers `*.log` et le fichier `.env`. Pour chercher les TODO du projet :

```bash
rg TODO
```

Sans chemin, rg cherche récursivement dans le répertoire courant. Dans un terminal, il regroupe les résultats par fichier et affiche les numéros de ligne :

```
tests/test_utils.py
5:    # TODO: tester une liste vide

README.md
3:TODO: compléter la documentation

static/app.min.js
1:function a(){return 1}/* TODO minifié */

static/app.js
8:// TODO: remplacer par fetch

src/utils.py
8:    # TODO: valider le schéma

src/api/routes.py
5:    # TODO: gérer l'authentification
```

Le TODO du dossier `node_modules` n'apparaît pas : nous verrons pourquoi dans la partie suivante. Attention, l'ordre des fichiers peut changer d'une exécution à l'autre, car rg cherche dans plusieurs fichiers en parallèle. Pour un ordre stable, `--sort path` trie les résultats par chemin, mais désactive le parallélisme.

On peut aussi limiter la recherche à un dossier ou à un fichier :

```bash
rg TODO src/
rg TODO src/utils.py
```

Sur un fichier unique, le nom du fichier n'est pas affiché : seuls le numéro de ligne et la ligne trouvée apparaissent.

Le motif est une expression régulière, avec la syntaxe de la bibliothèque `regex` de Rust, proche de celles de Perl ou de Python. Par exemple, pour trouver la définition d'une fonction, les imports d'un module ou un mot de passe écrit en dur :

```bash
rg 'def calculate_total'
rg '^from utils import|^import utils'
rg -i 'password\s*=\s*["\x27][^"\x27]{3,}'
```

La dernière commande trouve la ligne `password = "admin123"` du fichier `src/app.py`. Dans le motif, `\x27` désigne l'apostrophe, qu'on ne peut pas écrire telle quelle entre apostrophes dans le shell.

Comme grep, rg lit l'entrée standard quand on lui envoie des données par un pipe, par exemple `ps aux | rg python` ou `git log --oneline | rg -i fix`.

## Ce que rg ignore par défaut

Quand il parcourt un dossier, rg laisse de côté :
- les fichiers et dossiers exclus par un `.gitignore` (seulement à l'intérieur d'un dépôt git), par un `.ignore` ou par un `.rgignore` ;
- les fichiers et dossiers cachés, dont le nom commence par un point, comme `.git` ou `.env` ;
- les fichiers binaires.

Il ne suit pas non plus les liens symboliques, sauf avec l'option `-L`.

Une bonne partie de l'écart de vitesse avec `grep -r` vient de là. Sur un petit projet de test contenant 207 Mo de dépendances dans `node_modules`, `grep -rn TODO .` a mis 0,3 seconde et renvoyé 814 lignes, presque toutes dans `node_modules`, alors que `rg TODO` a renvoyé la seule ligne du code du projet en 6 ms. Avec les bonnes exclusions (`--exclude-dir=node_modules --exclude-dir=.git`), grep va d'ailleurs aussi vite.

À périmètre égal, rg garde l'avantage sur de plus gros volumes, grâce à son moteur de regex à base d'automates et à sa recherche en parallèle. Sur les 51 Mo d'en-têtes C de `/usr/include`, la même recherche a pris environ 20 ms avec rg, contre 50 ms avec `grep -rnI` (machine à 4 cœurs, cache disque chaud).

Notez que `node_modules` n'est pas exclu en dur : il l'est parce qu'il figure dans le `.gitignore`. Dans une copie du projet sans dossier `.git` (une archive décompressée par exemple), ce `.gitignore` n'est plus pris en compte et rg cherche de nouveau dans `node_modules`, sauf si on lui passe `--no-require-git`.

Pour élargir la recherche :

```bash
rg TODO --hidden        # Inclut les fichiers et dossiers cachés
rg TODO --no-ignore     # Ne tient plus compte des .gitignore, .ignore et .rgignore
rg TODO -uu             # Les deux à la fois
```

`-u` équivaut à `--no-ignore`, `-uu` y ajoute `--hidden`, et `-uuu` cherche aussi dans les fichiers binaires. Quand un résultat attendu n'apparaît pas, ajouter un ou deux `-u` est le moyen le plus rapide de savoir si le filtrage en est la cause.

Attention, `.git` n'est exclu que parce que c'est un dossier caché. Avec `--hidden`, rg cherche aussi dans les fichiers internes de git :

```bash
rg -l --hidden init
```

```
.git/logs/HEAD
.git/logs/refs/heads/master
src/app.py
.git/COMMIT_EDITMSG
```

On l'exclut alors explicitement avec un glob : `rg --hidden -g '!.git' init` ne renvoie plus que `src/app.py`.

Le fichier `.env` du projet est à la fois caché et ignoré par git : `rg DATABASE_URL --hidden` ne trouve rien, il faut `-uu`. Le plus simple est encore de le nommer, car un fichier passé explicitement en argument est toujours lu : `rg DATABASE_URL .env`.

Pour exclure des fichiers de vos recherches sans toucher au `.gitignore`, vous pouvez créer un fichier `.ignore` ou `.rgignore` à la racine du projet, avec la même syntaxe. rg le lit automatiquement, y compris en dehors d'un dépôt git :

```
*.min.js
*.min.css
dist/
coverage/
```

Enfin, `rg --files` liste les fichiers que rg fouillerait, sans rien chercher dedans. C'est pratique pour comprendre ce qui est filtré, ou pour compter les fichiers d'un type : `rg --files -t py | wc -l` affiche `4` sur notre projet.

## Casse, mots entiers et texte littéral

La recherche est sensible à la casse par défaut. `-i` la rend insensible, et `-S` (*smart case*) choisit tout seul : insensible à la casse si le motif est tout en minuscules, sensible dès qu'il contient une majuscule. `rg -S todo` trouve donc `TODO`, `Todo` et `todo`, alors que `rg -S Todo` ne trouve que `Todo`.

`-w` ne garde que les mots entiers :

```bash
rg -w total src/
```

```
src/utils.py
13:    total = 0
15:        total += item["price"] * item["quantity"]
17:    return total
```

`calculate_total` n'apparaît pas, car le `_` fait partie du mot.

`-F` cherche le texte tel quel, sans l'interpréter comme une regex, ce qui évite d'échapper les parenthèses et les crochets : `rg -F 'item["price"]'`. Et `-v` inverse la recherche, en affichant les lignes qui ne contiennent pas le motif.

## Filtrer par type de fichier ou par glob

`-t` limite la recherche à un type de fichier, et `-T` exclut un type :

```bash
rg TODO -t py           # Fichiers Python
rg TODO -t js -t ts     # JavaScript et TypeScript
rg TODO -T md           # Tout sauf le Markdown
```

Chaque type correspond à une liste de motifs de noms de fichiers, que `rg --type-list` affiche : `py` couvre `*.py` et `*.pyi`, `js` couvre aussi `*.jsx`, `*.mjs` ou `*.vue`. ripgrep 14.1.0 connaît plus de 200 types.

`--type-add` définit un type supplémentaire :

```bash
rg TODO --type-add 'web:*.{html,css,js}' -t web
```

Choisissez un nom qui n'existe pas déjà : si le type existe, `--type-add` ajoute les motifs à sa définition au lieu de la remplacer. Le type `config`, par exemple, existe et couvre déjà `*.cfg`, `*.conf`, `*.config` et `*.ini`.

Pour un filtrage plus fin, `-g` prend un glob, avec la même syntaxe que les `.gitignore`. Un `!` devant le glob exclut les fichiers correspondants :

```bash
rg TODO -g '*.{py,md}'              # Seulement les fichiers .py et .md
rg TODO -g '!*.min.js'              # Tout sauf les fichiers minifiés
rg TODO -g '!tests' -g '!vendor'    # Sans les dossiers tests et vendor
```

Pensez aux apostrophes autour du glob, sinon le shell risque de l'interpréter avant rg. Attention aussi, un glob passé avec `-g` l'emporte sur toutes les autres règles d'exclusion : `rg DATABASE_URL -g '*.env'` trouve bien le fichier `.env`, qui est pourtant caché et ignoré par git.

## Contexte et format de la sortie

`-A`, `-B` et `-C` affichent des lignes autour de chaque résultat, respectivement après, avant et des deux côtés :

```bash
rg -C 2 FIXME
```

```
src/utils.py
14-    for item in items:
15-        total += item["price"] * item["quantity"]
16:    # FIXME: arrondir au centime
17-    return total
```

Les lignes de contexte sont marquées d'un `-`, la ligne trouvée d'un `:`.

D'autres options changent ce qui est affiché :

```bash
rg -l TODO                       # Seulement les noms des fichiers qui contiennent le motif
rg --files-without-match TODO    # Les fichiers qui ne le contiennent pas
rg -c TODO                       # Le nombre de lignes trouvées par fichier
rg --count-matches TODO          # Le nombre d'occurrences par fichier
rg -o 'TODO: \w+'                # Seulement la partie qui correspond au motif
rg -N TODO                       # Sans les numéros de ligne
rg -I TODO                       # Sans les noms de fichiers
```

La différence entre `-c` et `--count-matches` se voit dès qu'une ligne contient plusieurs fois le motif : `rg -c item src/utils.py` affiche `3` (lignes) et `rg --count-matches item src/utils.py` affiche `5` (occurrences). Pour un total sur tout le projet, on additionne les compteurs avec awk : `rg --count-matches total | awk -F: '{sum+=$NF} END {print sum}'`.

Enfin, `--json` produit un objet JSON par ligne (début de fichier, résultat, fin de fichier, statistiques), facile à exploiter dans un script ou une CI avec [jq]({% post_url 2025-09-17-Comment-utiliser-jq %}) :

```bash
rg TODO --json | jq -r 'select(.type == "match") | "\(.data.path.text):\(.data.line_number)"'
```

```
tests/test_utils.py:5
README.md:3
static/app.min.js:1
static/app.js:8
src/utils.py:8
src/api/routes.py:5
```

## Prévisualiser un remplacement

`-r` remplace le texte trouvé dans la sortie, sans jamais modifier les fichiers. Les groupes capturés sont disponibles avec `$1`, `$2`, etc. :

```bash
rg 'log\((\w+)\)' -r 'logger.info($1)'
```

```
static/app.js
9:logger.info(message);
```

Une fois le résultat vérifié, on applique le remplacement avec [sed]({% post_url 2026-01-19-Sed-editer-des-fichiers-en-ligne-de-commande %}), sur les fichiers que `rg -l` liste. Par exemple, pour renommer une fonction :

```bash
rg -l 'old_function' | xargs sed -i 's/old_function/new_function/g'
```

Si des chemins peuvent contenir des espaces, il faut séparer les noms de fichiers par un caractère nul : `rg -l -0 'old_function' | xargs -0 sed -i 's/old_function/new_function/g'`. Sur macOS, avec le sed de BSD, l'option s'écrit `sed -i ''`.

## Chercher sur plusieurs lignes

Par défaut, rg cherche ligne par ligne. Avec `-U` (*multiline*), un motif peut s'étendre sur plusieurs lignes. Par exemple, pour trouver les fonctions JavaScript vides, y compris quand l'accolade fermante est sur la ligne suivante :

```bash
rg -U 'function \w+\([^)]*\)\s*\{\s*\}' -t js
```

```
static/app.js
3:function noop() {}
5:function aFaire() {
6:}
```

Sans `-U`, seule `noop` est trouvée.

Attention, même avec `-U`, le point ne correspond pas à un retour à la ligne. Pour cela, il faut ajouter `--multiline-dotall`, ou le drapeau `(?s)` au début du motif. Sans `--multiline-dotall`, la commande suivante ne trouverait rien. Avec cette option, elle trouve le bloc `try` de `src/app.py` :

```bash
rg -U --multiline-dotall 'try:.*?except' -t py
```

```
src/app.py
11:        try:
12:            print(self.config)
13:        except KeyError:
```

## Les regex PCRE2

Le moteur de regex par défaut de rg ne gère ni les *lookarounds* ni les références arrière. C'est un choix : il repose sur des automates finis, ce qui garantit un temps de recherche linéaire. Quand on en a besoin, `-P` bascule sur le moteur PCRE2 :

```bash
rg -P '(?<=def )\w+(?=\()' -t py -o    # Noms des fonctions Python
rg -P 'error(?!.*404)' app.log         # Lignes avec "error" mais sans "404"
rg -P '\b(\w+)\s+\1\b'                 # Mots répétés, comme "the the"
```

Avec `-o`, la première commande n'affiche que les noms de fonctions :

```
tests/test_utils.py
4:test_calculate_total

src/utils.py
4:load_config
12:calculate_total

src/api/routes.py
4:get_total

src/app.py
7:__init__
10:run
```

Sans `-P`, rg refuse par exemple le troisième motif, et propose la solution :

```
rg: regex parse error:
    (?:\b(\w+)\s+\1\b)
                 ^^
error: backreferences are not supported

Consider enabling PCRE2 with the --pcre2 flag, which can handle backreferences
and look-around.
```

PCRE2 est une option de compilation de ripgrep : `rg --version` indique si elle est disponible. C'est le cas du paquet d'Ubuntu 24.04 utilisé ici (`PCRE2 10.42 is available`) et, d'après la FAQ du projet, de la plupart des binaires publiés sur GitHub.

## Le fichier de configuration

ripgrep ne cherche pas de fichier de configuration à un emplacement prédéfini. Il faut créer un fichier (le nom et l'emplacement sont libres) et indiquer son chemin dans la variable d'environnement `RIPGREP_CONFIG_PATH`, par exemple dans le `~/.bashrc` :

```bash
export RIPGREP_CONFIG_PATH="$HOME/.ripgreprc"
```

Le fichier contient une option par ligne, sans guillemets ni échappement, et les lignes qui commencent par `#` sont des commentaires :

```bash
# ~/.ripgreprc

# Smart case par défaut
--smart-case

# Ignorer les fichiers minifiés
--glob=!*.min.js
--glob=!*.min.css

# Chercher aussi dans les fichiers cachés, mais jamais dans .git
--hidden
--glob=!.git

# Un type "web" pour les fichiers du front
--type-add=web:*.{html,css,js}
```

Une option qui prend une valeur s'écrit avec un `=` (`--glob=!*.min.js`), ou sur deux lignes. Les options passées en ligne de commande sont ajoutées après celles du fichier, et l'emportent donc en cas de conflit. `--no-config` ignore complètement le fichier, et `--debug` indique celui qui a été chargé.

## Utiliser rg avec fzf, Vim, VS Code et git

Avec [fzf](https://github.com/junegunn/fzf), on obtient une recherche interactive. `rg --files | fzf` permet de choisir un fichier du projet. Pour chercher dans le contenu, avec un aperçu du fichier, on peut ajouter cette petite fonction dans le `~/.bashrc` :

```bash
rgf() {
  rg --line-number --no-heading --color=always "$@" | fzf --ansi \
    --delimiter ':' \
    --preview 'bat --color=always {1} --highlight-line {2}'
}
```

`--no-heading` met le nom du fichier sur chaque ligne (`chemin:ligne:contenu`), ce qui permet à fzf de récupérer le chemin (`{1}`) et le numéro de ligne (`{2}`) pour l'aperçu affiché par [bat](https://github.com/sharkdp/bat). Sur Debian et Ubuntu, la commande `bat` s'appelle `batcat`. On lance ensuite `rgf TODO`.

Dans Vim ou Neovim, rg peut remplacer grep pour la commande `:grep` :

```vim
" .vimrc / init.vim
set grepprg=rg\ --vimgrep\ --smart-case
set grepformat=%f:%l:%c:%m

" Chercher le mot sous le curseur
nnoremap <leader>g :grep <C-R><C-W><CR>
```

`--vimgrep` affiche chaque résultat sous la forme `fichier:ligne:colonne:texte`, le format décrit par `grepformat`.

VS Code utilise déjà ripgrep pour sa recherche dans les fichiers : il n'y a rien à installer. Son comportement se règle avec les paramètres `search.exclude`, `search.useIgnoreFiles` (prise en compte des `.gitignore`, activée par défaut) ou `search.smartCase` (l'équivalent de `-S`, désactivé par défaut).

Avec git, on peut limiter la recherche aux fichiers modifiés depuis le dernier commit, ou à ceux d'un commit donné :

```bash
rg TODO $(git diff --name-only HEAD)
git show --name-only --format= HEAD | xargs rg TODO
```

rg ne cherche que dans les fichiers présents sur le disque. Pour retrouver du code supprimé, il faut passer par l'historique : `git log -S "old_function" --source --all --oneline` liste les commits qui ont ajouté ou supprimé cette chaîne.

## Voir aussi

- [Sed : éditer des fichiers en ligne de commande avec des regex]({% post_url 2026-01-19-Sed-editer-des-fichiers-en-ligne-de-commande %})
- [Comment manipuler du JSON en ligne de commande avec jq]({% post_url 2025-09-17-Comment-utiliser-jq %})
- [Le guide utilisateur de ripgrep](https://github.com/BurntSushi/ripgrep/blob/master/GUIDE.md)
- [La FAQ de ripgrep](https://github.com/BurntSushi/ripgrep/blob/master/FAQ.md)
