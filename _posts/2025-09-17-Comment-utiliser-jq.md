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

- Installer rapidement `jq` (Linux/macOS/Windows)
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

Nous utiliserons ces exemples JSON (fichier `data.json`) dans plusieurs exemples :

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

## Bases en 60 secondes

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

- `-C` force les couleurs, `-M` les désactive.

### 2) Extraire un champ simple (avec sortie brute)

```bash
jq -r '.users[0].name' data.json      # Alice
jq -r '.users[] | select(.id==2) | .name' data.json   # Bob
```

- Utilisez `-r` pour éviter les guillemets dans la sortie texte.

### 3) Lister les noms de tous les utilisateurs

```bash
jq -r '.users[].name' data.json
```

### 4) Filtrer sur une condition (select)

```bash
jq -r '.users[] | select(.active) | .name' data.json   # actifs uniquement
jq '.users[] | select(.score >= 30)' data.json         # objets complets
```

### 5) Formater des lignes personnalisées

```bash
jq -r '.users[] | "\(.id)\t\(.name)\tactive=\(.active)"' data.json
```

### 6) Trier (sort_by) et inverser

```bash
jq '.users | sort_by(.score)' data.json
jq -r '.users | sort_by(.created_at) | reverse | .[].name' data.json
```

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

### 8) Valeurs uniques (unique, unique_by)

```bash
jq '.users | map(.tags) | add | unique' data.json              # tags uniques
jq '.users | unique_by(.active) | map(.name)' data.json        # un par statut
```

### 9) Groupement et comptage (group_by + length)

```bash
jq '.users | group_by(.active) | map({active: .[0].active, count: length})' data.json
```

### 10) Changer la stucture du json (projection)

```bash
jq '.users | map({id, name, score})' data.json
jq '.users | map({label: .name, meta: {id, active}})' data.json
```

### 11) Mettre à jour des champs (update)

```bash
# Augmenter tous les scores de 10%
jq '.users |= map(.score = (.score * 1.10))' data.json
# Marquer Bob actif
jq '.users |= map(if .name=="Bob" then .active=true else . end)' data.json
```

### 12) Concaténer/assembler des tableaux (plusieurs fichiers)

```bash
# Concat simple (deux fichiers contenant des tableaux JSON)
jq -s 'add' a.json b.json
# Accumuler tous les inputs (flux)
cat a.json b.json | jq -s 'flatten'     # aplatit un niveau
```

### 13) Extraire seulement certaines clés

```bash
jq '.users | map({id, name})' data.json
jq '.users | map({id, name, tags: (.tags | join(","))})' data.json
```

### 14) JSON Lines (une ligne par objet) et compact

```bash
# Sortie compacte (-c) et une ligne par élément
echo '{"items":[{"id":1},{"id":2}]}' | jq -c '.items[]'
# Repasser en tableau structuré
printf '%s
' '{"id":1}' '{"id":2}' | jq -s '.'
```

### 15) Variables depuis la CLI (`--arg`, `--argjson`)

```bash
# --arg crée une variable chaîne
jq -r --arg user "Alice" '.users[] | select(.name==$user) | .id' data.json
# --argjson prend du JSON réel (nombre, booléen, objet)
jq --argjson threshold 30 '.users | map(select(.score > $threshold))' data.json
```

---

## Bonus: combiner avec curl, docker, kubectl

```bash
# APIs HTTP
curl -s https://api.github.com/repos/owner/repo/issues | jq -r '.[].title'
# Docker (inspect)
docker inspect --format='{{json .State.Health}}' fastapi-app | jq
# Kubernetes (objets pods)
kubectl get pods -o json | jq -r '.items[] | "\(.metadata.name)\t\(.status.phase)"'
```

> Voir aussi: votre article Docker : {% post_url 2025-08-16-Comment-dockeriser-une-api-web-avec-FastAPI %}

---

## Options essentielles à connaître

- `-r` raw-output: sorties texte sans guillemets (parfait pour scripts)
- `-c` compact-output: une ligne par objet (idéal pour logs/JSONL)
- `-S` sort-keys: clés triées (diffs plus stables)
- `-M` monochrome-output: désactiver les couleurs
- `-e` exit-status: code 0 si filtre produit une sortie non nulle/vraie
- `-s` slurp: lit toutes les entrées et les met dans un tableau
- `-n` null-input: démarre sans entrée (pour générer du JSON de zéro)

Exemple avec `-n`:

```bash
jq -n '{now: now | todateiso8601}'
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
