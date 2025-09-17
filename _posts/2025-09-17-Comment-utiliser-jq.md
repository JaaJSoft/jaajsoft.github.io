---
layout: article
title: "Comment utiliser jq : 15 exemples indispensables pour manipuler du JSON en ligne de commande"
author: Pierre Chopinet
tags:
  - linux
  - jq
  - json
  - cli
  - shell
  - outils
  - tutoriel
---

`jq` est un couteau suisse pour lire, filtrer et transformer du JSON en ligne de commande. Il s’intègre parfaitement avec `curl`, `kubectl`, `docker`, des logs JSON, etc.
<!--more-->

Objectifs de l’article :

- Installer `jq` (Linux/macOS/Windows)
- Comprendre les bases (filtres, pipe, tableaux)
- Appliquer 15 process concrets (extractions, filtres, agrégations, tri, mise à jour, concat, variables…)
- Connaître les options essentielles (`-r`, `-c`, `-S`, `-e`, `--arg`, `--argjson`)

---

## Installation rapide

- Debian/Ubuntu: `sudo apt install jq`
- Fedora: `sudo dnf install jq`
- Arch: `sudo pacman -S jq`
- macOS (Homebrew): `brew install jq`
- Windows (Scoop): `scoop install jq`
- Windows (Chocolatey): `choco install jq`
- Docker (sans installer localement): `docker run --rm -i imega/jq jq .`

---

## Données d’exemple

Nous utiliserons l'exemple JSON ci-dessous (fichier `data.json`) :

```json
{
  "users": [
    {"id": 1, "name": "Alice", "active": true,  "tags": ["admin", "ops"], "score": 42.5, "created_at": "2025-09-01T12:00:00Z"},
    {"id": 2, "name": "Bob",   "active": false, "tags": ["dev"],          "score": 12.1, "created_at": "2025-08-29T10:30:00Z"},
    {"id": 3, "name": "Chloé", "active": true,  "tags": ["dev", "ops"],  "score": 31.7, "created_at": "2025-09-05T08:45:00Z"}
  ]
}
```

---

## Les bases de jq

- Filtre identité : `.` renvoie l’entrée telle quelle.
- Accéder à un champ : `.users`, puis `.users[0]`, `.users[].name`.
- Chaîner les filtres : `.users[] | select(.active == true) | .name`.
- Construire un objet : `{id: .id, label: .name}`.
- Interpolation de chaînes : `"\(.name) — id=\(.id)"`.

> Option `-r` (raw) : sort des chaînes brutes sans guillemets. Très pratique pour des boucles shell.

---

## 15 commandes jq qui changent la vie

### 1) Pretty‑print et validation rapide

```bash
cat data.json | jq .            # mise en forme + couleurs
cat data.json | jq -e . >/dev/null && echo OK || echo KO  # -e: code de sortie selon validité
```

Sortie :

- Mise en forme du JSON :
```json
{
  "users": [
    {"id": 1, "name": "Alice", "active": true,  "tags": ["admin", "ops"], "score": 42.5, "created_at": "2025-09-01T12:00:00Z"},
    {"id": 2, "name": "Bob",   "active": false, "tags": ["dev"],          "score": 12.1, "created_at": "2025-08-29T10:30:00Z"},
    {"id": 3, "name": "Chloé", "active": true,  "tags": ["dev", "ops"],  "score": 31.7, "created_at": "2025-09-05T08:45:00Z"}
  ]
}
```
- Validation :
```text
OK
```

Explications :
- `jq .` applique le filtre identité pour pretty‑printer le JSON d’entrée.
- Avec `-e`, jq renvoie un code de sortie 0 si l’entrée est un JSON valide (d’où "OK").

- `-C` force les couleurs, `-M` les désactive.

### 2) Extraire un champ simple

```bash
jq -r '.users[0].name' data.json      # Alice
jq -r '.users[] | select(.id==2) | .name' data.json   # Bob
```

Sortie :
```text
Alice
Bob
```

Explication :
- `select(.id==2)` filtre l’élément voulu, `-r` supprime les guillemets autour des chaînes.

### 3) Lister les noms de tous les utilisateurs

```bash
jq -r '.users[].name' data.json
```

Sortie :
```text
Alice
Bob
Chloé
```

### 4) Filtrer sur une condition (select)

```bash
jq -r '.users[] | select(.active) | .name' data.json   # actifs uniquement
jq '.users[] | select(.score >= 30)' data.json         # objets complets
```

Sortie :
- Noms des actifs :
```text
Alice
Chloé
```
- Objets avec score >= 30 :
```text
{
  "id": 1,
  "name": "Alice",
  "active": true,
  "tags": ["admin", "ops"],
  "score": 42.5,
  "created_at": "2025-09-01T12:00:00Z"
}
{
  "id": 3,
  "name": "Chloé",
  "active": true,
  "tags": ["dev", "ops"],
  "score": 31.7,
  "created_at": "2025-09-05T08:45:00Z"
}
```

Explication :
- `select(expr)` laisse passer uniquement les éléments pour lesquels l’expression est vraie.

### 5) Formater des lignes personnalisées

```bash
jq -r '.users[] | "\(.id)\t\(.name)\tactive=\(.active)"' data.json
```

Sortie :
```text
1	Alice	active=true
2	Bob	active=false
3	Chloé	active=true
```

> Astuce : Les interpolations `\( ... )` insèrent des valeurs dans une chaîne & `-r` évite les guillemets et garde les tabulations.

### 6) Trier (sort_by) et inverser

```bash
jq '.users | sort_by(.score)' data.json
jq -r '.users | sort_by(.created_at) | reverse | .[].name' data.json
```

Sortie :
- Tri par score (ascendant) :
```json
[
  {"id": 2, "name": "Bob",   "active": false, "tags": ["dev"],          "score": 12.1, "created_at": "2025-08-29T10:30:00Z"},
  {"id": 3, "name": "Chloé", "active": true,  "tags": ["dev", "ops"],  "score": 31.7, "created_at": "2025-09-05T08:45:00Z"},
  {"id": 1, "name": "Alice", "active": true,  "tags": ["admin", "ops"], "score": 42.5, "created_at": "2025-09-01T12:00:00Z"}
]
```
- Noms triés par date (récents d’abord) :
```text
Chloé
Alice
Bob
```

> Note : `reverse` inverse l’ordre après le tri croissant par `created_at`.

### 7) Sommes, min/max, moyenne

```bash
# somme des scores
jq '[.users[].score] | add' data.json
# min / max
jq 'min_by(.users[].score)?' data.json   # pas idéal; préférez:
jq '.users | min_by(.score)' data.json
jq '.users | max_by(.score)' data.json
# moyenne approximative
jq '[.users[].score] | add / length' data.json
```

Sortie :
- Somme :
```text
86.3
```
- min_by(.users[].score)? (à éviter) :
```text
null
```
- min / max corrects :
```text
{"id": 2, "name": "Bob",   "active": false, "tags": ["dev"],          "score": 12.1, "created_at": "2025-08-29T10:30:00Z"}
{"id": 1, "name": "Alice", "active": true,  "tags": ["admin", "ops"], "score": 42.5, "created_at": "2025-09-01T12:00:00Z"}
```
- Moyenne (~) :
```text
28.766666666666666
```

> Note : Préférez toujours `.users | min_by(.score)`/`max_by(.score)` sur le tableau plutôt que d’essayer d’y accéder depuis la racine.

### 8) Valeurs uniques (unique, unique_by)

```bash
jq '.users | map(.tags) | add | unique' data.json              # tags uniques
jq '.users | unique_by(.active) | map(.name)' data.json        # un par statut
```

Sortie :
- Tags uniques :
```json
["admin", "dev", "ops"]
```
- Un nom par statut (actif/inactif) :
```json
["Alice", "Bob"]
```

> Note : `unique`/`unique_by` dédupliquent. Ici, on garde un seul utilisateur par statut actif/inactif.

### 9) Groupement et comptage (group_by + length)

```bash
jq '.users | group_by(.active) | map({active: .[0].active, count: length})' data.json
```

Sortie :
```json
[{"active": false, "count": 1}, {"active": true, "count": 2}]
```

> Note : `group_by` regroupe par clé (il trie par la clé avant de grouper).

### 10) Changer la structure du json (projection)

```bash
jq '.users | map({id, name, score})' data.json
jq '.users | map({label: .name, meta: {id, active}})' data.json
```

Sortie :
- Projection simple :
```json
[
  {"id":1, "name":"Alice", "score":42.5},
  {"id":2, "name":"Bob",   "score":12.1},
  {"id":3, "name":"Chloé", "score":31.7}
]
```
- Projection imbriquée :
```json
[
  {"label":"Alice", "meta":{"id":1, "active":true}},
  {"label":"Bob",   "meta":{"id":2, "active":false}},
  {"label":"Chloé", "meta":{"id":3, "active":true}}
]
```

Ces projections permettent d’extraire/alléger les payloads pour logs, exports, etc.

### 11) Mettre à jour des champs (update)

```bash
# Augmenter tous les scores de 10%
jq '.users |= map(.score = (.score * 1.10))' data.json
# Marquer Bob actif
jq '.users |= map(if .name=="Bob" then .active=true else . end)' data.json
```

Sortie :
- Scores +10% :
```json
{
  "users": [
    {"id": 1, "name": "Alice", "active": true,  "tags": ["admin", "ops"], "score": 46.75, "created_at": "2025-09-01T12:00:00Z"},
    {"id": 2, "name": "Bob",   "active": false, "tags": ["dev"],          "score": 13.31, "created_at": "2025-08-29T10:30:00Z"},
    {"id": 3, "name": "Chloé", "active": true,  "tags": ["dev", "ops"],  "score": 34.87, "created_at": "2025-09-05T08:45:00Z"}
  ]
}
```
- Bob marqué actif :
```json
{
  "users": [
    {"id": 1, "name": "Alice", "active": true,  "tags": ["admin", "ops"], "score": 42.5, "created_at": "2025-09-01T12:00:00Z"},
    {"id": 2, "name": "Bob",   "active": true,  "tags": ["dev"],          "score": 12.1, "created_at": "2025-08-29T10:30:00Z"},
    {"id": 3, "name": "Chloé", "active": true,  "tags": ["dev", "ops"],  "score": 31.7, "created_at": "2025-09-05T08:45:00Z"}
  ]
}
```

> Astuce : L’opérateur `|=` met à jour le champ ciblé. Utilisez `if/then/else` pour modifier conditionnellement.

### 12) Concaténer/assembler des tableaux (plusieurs fichiers)

```bash
# Concat simple (deux fichiers contenant des tableaux JSON)
jq -s 'add' a.json b.json
# Accumuler tous les inputs (flux)
cat a.json b.json | jq -s 'flatten'     # aplatit un niveau
```

En entrée :
- a.json = `[1, 2]`
- b.json = `[3, 4]`

Sortie :
- Concat (`-s add`) :
```json
[1, 2, 3, 4]
```
flatten :
```text
[1, 2, 3, 4]
```

> Astuce : `-s` (slurp) lit tous les fichiers/entrées et crée un tableau d’entrées avant d’appliquer le filtre.

### 13) Extraire seulement certaines clés

```bash
jq '.users | map({id, name})' data.json
jq '.users | map({id, name, tags: (.tags | join(","))})' data.json
```

Sortie :
- Sélection de clés :
```json
[
  {"id":1, "name":"Alice"},
  {"id":2, "name":"Bob"},
  {"id":3, "name":"Chloé"}
]
```
- Avec concaténation des tags :
```json
[
  {"id":1, "name":"Alice", "tags":"admin,ops"},
  {"id":2, "name":"Bob",   "tags":"dev"},
  {"id":3, "name":"Chloé", "tags":"dev,ops"}
]
```

> Astuce : `join(",")` fusionne un tableau de chaînes en une seule chaîne.

### 14) JSON Lines (une ligne par objet) et compact

```bash
# Sortie compacte (-c) et une ligne par élément
echo '{"items":[{"id":1},{"id":2}]}' | jq -c '.items[]'
# Repasser en tableau structuré
printf '%s
' '{"id":1}' '{"id":2}' | jq -s '.'
```

Sortie :
- JSONL compact :
```text
{"id":1}
{"id":2}
```
- Slurp (`-s`) -> tableau :
```json
[
  {"id": 1},
  {"id": 2}
]
```

> Astuce : `-c` est idéal pour logs/streams (une ligne par objet). `-s` (slurp) recompose un tableau.

### 15) Variables depuis la CLI (`--arg`, `--argjson`)

```bash
# --arg crée une variable chaîne
jq -r --arg user "Alice" '.users[] | select(.name==$user) | .id' data.json
# --argjson prend du JSON réel (nombre, booléen, objet)
jq --argjson threshold 30 '.users | map(select(.score > $threshold))' data.json
```

Sortie :
- Variable chaîne :
```text
1
```
- Variable JSON (seuil=30) :
```json
[
  {"id": 1, "name": "Alice", "active": true,  "tags": ["admin", "ops"], "score": 42.5, "created_at": "2025-09-01T12:00:00Z"},
  {"id": 3, "name": "Chloé", "active": true,  "tags": ["dev", "ops"],  "score": 31.7, "created_at": "2025-09-05T08:45:00Z"}
]
```

Explication :
- `--arg name value` passe une chaîne; `--argjson name value` passe une valeur JSON typée accessible via `$name`.

---

## Bonus : combiner avec curl, docker, kubectl

```bash
# APIs HTTP
curl -s https://api.github.com/repos/owner/repo/issues | jq -r '.[].title'
# Docker (inspect)
docker inspect --format='{{json .State.Health}}' fastapi-app | jq
# Kubernetes (objets pods)
kubectl get pods -o json | jq -r '.items[] | "\(.metadata.name)\t\(.status.phase)"'
```

> Voir aussi : {% post_url 2025-08-16-Comment-dockeriser-une-api-web-avec-FastAPI %}

---

## Options essentielles à connaître

- `-r` raw-output : sorties texte sans guillemets (parfait pour scripts)
- `-c` compact-output : une ligne par objet (idéal pour logs/JSONL)
- `-S` sort-keys : clés triées (diffs plus stables)
- `-M` monochrome-output : désactiver les couleurs
- `-e` exit-status : code 0 si filtre produit une sortie non nulle/vraie
- `-s` slurp : lit toutes les entrées et les met dans un tableau
- `-n` null-input : démarre sans entrée (pour générer du JSON de zéro)

```

---

## Pièges et bonnes pratiques

- Toujours penser à `-r` si vous attendez du texte (sinon vous aurez des quotes).
- Préférez `select(...)` plutôt que des `if` imbriqués quand vous filtrez.
- Pour de gros volumes, utilisez `-c` et évitez les pretty‑prints inutiles qui peuvent ralentir jq.
- Utilisez `--arg` / `--argjson` plutôt que de bidouiller des chaînes JSON en shell.
- Pour déboguer un pipeline, insérez `| debug` ou `. as $x | ... | $x`.

---

## Cheatsheet

- Accéder : `.foo`, `.foo[0]`, `.foo[]`, `.foo.bar?` (optionnel)
- Filtrer : `select(.a==1 and .b>0)`, `map(...)`, `any`, `all`
- Trier : `sort_by(.key) | reverse`
- Grouper : `group_by(.key) | map({key: .[0].key, count: length})`
- Agréger : `add`, `length`, `unique`, `unique_by(.k)`
- Construire : `{a: .x, b: (.y | tostring)}`
- Variables : `--arg name val`, `--argjson k '{"a":1}'`

---

## Conclusion

`jq` est indispensable pour trier, filtrer, agréger et reformater du JSON sans écrire un script. Gardez cette page sous la main, et n’hésitez pas à adapter les recettes à vos données..
