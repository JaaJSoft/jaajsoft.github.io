---
layout: article
title: "Comment transformer un JSON en CSV avec jq"
author: Pierre Chopinet
tags:
  - linux
  - jq
  - json
  - csv
  - cli
  - data
  - outils
  - shell
  - tutoriel
---

Convertir rapidement des données JSON en CSV est un besoin récurrent pour analyser dans Excel/Google Sheets, charger dans un datawarehouse ou partager un rapport simple.
Avec jq, vous pouvez produire un CSV propre en une seule commande, sans écrire de script.
<!--more-->

Objectifs de l’article :
- Comprendre les primitives jq utiles pour le CSV (`@csv`, `@tsv`, `-r`, sélection des champs)
- Générer un CSV depuis un tableau d’objets JSON (avec ou sans en‑tête)
- Gérer les champs imbriqués, les tableaux, les valeurs nulles et les types
- Traiter du JSON Lines (NDJSON) et de gros volumes
- Éviter les pièges fréquents (ordre des colonnes, guillemets, séparateurs)

> Si vous débutez avec jq, commencez par l’article précédent : [Comment manipuler du JSON en ligne de commande avec jq]({% post_url 2025-09-17-Comment-utiliser-jq %}).

---

## Données d’exemple

Fichier `users.json` :

```json
[
  {"id": 1, "name": "Alice", "active": true,  "tags": ["admin", "ops"], "score": 42.5, "created_at": "2025-09-01T12:00:00Z"},
  {"id": 2, "name": "Bob",   "active": false, "tags": ["dev"],          "score": 12.1, "created_at": "2025-08-29T10:30:00Z"},
  {"id": 3, "name": "Chloé", "active": true,  "tags": ["dev", "ops"],  "score": 31.7, "created_at": "2025-09-05T08:45:00Z"}
]
```

---

## 1) CSV le plus simple avec colonnes explicites

La méthode la plus robuste est d’énumérer explicitement les colonnes et leur ordre.

```bash
jq -r '.[] | [.id, .name, .active, .score] | @csv' users.json
```

Sortie :
```text
1,"Alice",true,42.5
2,"Bob",false,12.1
3,"Chloé",true,31.7
```

Explications :
- `.[]` parcourt chaque objet du tableau.
- `[.id, .name, .active, .score]` construit un tableau de champs, dans l’ordre voulu.
- `@csv` sérialise en CSV correct (échappement des guillemets, etc.).
- `-r` (raw) sort une chaîne brute sans guillemets JSON autour de la ligne.

> Avantage : contrôle total de l’ordre et du sous-ensemble de colonnes.

---

## 2) Ajouter une ligne d’en‑tête

Deux approches :

- Entête statique (facile)

```bash
{
  printf 'id,name,active,score\n';
  jq -r '.[] | [.id, .name, .active, .score] | @csv' users.json;
} > users.csv
```

- Entête dynamique à partir des clés du premier objet

```bash
jq -r '(
  .[0] | keys_unsorted
) as $keys
| $keys,
  (.[] | [ .[ $keys[] ] ])
| @csv' users.json
```

> Note : `keys_unsorted` lit les clés telles qu’elles apparaissent (pas triées). Selon vos données, l’ordre peut varier; pour un ordre stable, préférez une liste explicite de colonnes.

---

## 3) Champs imbriqués et tableaux

- Extraire une propriété imbriquée (avec valeur optionnelle)

```bash
jq -r '.[] | [.id, .name, (.profile.city // ""), .score] | @csv' users.json
```

- Convertir un tableau en chaîne séparée par `;`

```bash
jq -r '.[] | [.id, .name, (.tags | join(";")), .score] | @csv' users.json
```

- Formater/convertir les types explicitement

```bash
jq -r '.[] | [(.id|tostring), .name, (.active|tostring), (.score // 0)] | @csv' users.json
```

---

## 4) Filtrer, trier, sélectionner des colonnes

- Uniquement les utilisateurs actifs, triés par score décroissant, colonnes réduites :

```bash
jq -r '.
  | map(select(.active))
  | sort_by(.score) | reverse
  | .[] | [.id, .name, .score]
  | @csv' users.json
```

- Renommer des colonnes via l’entête statique :

```bash
{
  printf 'user_id,full_name,score\n';
  jq -r '.[] | [.id, .name, .score] | @csv' users.json;
} > users.csv
```

---

## 5) JSON Lines (NDJSON)

Si vous avez un fichier NDJSON (`users.jsonl`) avec un objet JSON par ligne :

```text
{"id":1,"name":"Alice","active":true,"score":42.5}
{"id":2,"name":"Bob","active":false,"score":12.1}
{"id":3,"name":"Chloé","active":true,"score":31.7}
```

Deux patterns :

- Avec en‑tête statique

```bash
{
  printf 'id,name,active,score\n';
  jq -r '[.id, .name, .active, .score] | @csv' users.jsonl;
} > users.csv
```

- En‑tête dynamique (déduit de la première ligne)

```bash
jq -Rns '  # -R: lire brut, -n: pas d’entrée par défaut, -s: slurp
  def fromjsonl: split("\n") | map(select(length>0) | fromjson);
  (fromjsonl) as $rows
  | ($rows[0] | keys_unsorted) as $keys
  | $keys,
    ($rows[] | [ .[ $keys[] ] ])
  | @csv' users.jsonl
```

> Pour de très gros fichiers NDJSON, préférez l’en‑tête statique et un traitement ligne à ligne (sans charger tout en mémoire).

---

## 6) TSV plutôt que CSV (quand c’est pertinent)

Le TSV est pratique quand vos valeurs contiennent souvent des virgules. Il s’aligne bien en terminal.

```bash
jq -r '.[] | [.id, .name, .active, .score] | @tsv' users.json > users.tsv
```

Aperçu lisible en colonne (Linux/macOS) :

```bash
column -t -s $'\t' users.tsv
```

> Sous Windows PowerShell, utilisez `Format-Table` ou importez le TSV dans Excel.

---

## 7) Gérer NULL, valeurs manquantes et formats

- Valeurs manquantes -> vide

```bash
jq -r '.[] | [.id, (.email // ""), (.score // 0)] | @csv' users.json
```

- Forcer un formatage (ex : arrondir à 2 décimales)

```bash
jq -r '.[] | [.id, .name, ( (.score // 0) | tonumber | (.*100|round/100) )] | @csv' users.json
```

- Dates ISO -> date seule

```bash
jq -r '.[] | [.id, (.created_at | split("T")[0])] | @csv' users.json
```

---

## 8) Gros volumes : streamer sans tout charger

- Tableau massif : évitez `map(...)` si possible; parcourez en flux :

```bash
jq -r '.[] | [.id, .name, .score] | @csv' huge.json > out.csv
```

- NDJSON massif : traitez ligne à ligne, entête statique :

```bash
{
  printf 'id,name,score\n';
  jq -r '[.id, .name, .score] | @csv' huge.jsonl;
} > out.csv
```

> Astuce : ajoutez `pv` (Linux/macOS) dans le pipeline pour suivre la progression : `jq ... | pv > out.csv`.

---

## Options jq à retenir pour manipuler des CSV

- `-r` raw‑output : indispensable pour obtenir des lignes sans guillemets JSON externes
- `@csv` / `@tsv` : sérialisation sûre des valeurs (échappement, quoting)
- `keys_unsorted` : récupérer les noms de colonnes depuis un objet
- `//` (coalescence) : définir une valeur par défaut pour NULL/absent
- `tostring` / `tonumber` : conversions explicites

---

## Pièges et bonnes pratiques

- Ne faites pas confiance à l’ordre de `keys` si vos objets ont des clés différentes; définissez vos colonnes explicitement pour des exports stables.
- Utilisez toujours `-r` avec `@csv/@tsv` pour éviter les guillemets JSON autour des lignes.
- Si vos valeurs contiennent des virgules, sauts de ligne ou guillemets, laissez jq gérer l’échappement via `@csv` plutôt que de concaténer des chaînes vous‑même.
- Préférez `@tsv` quand vous regardez le résultat en terminal; convertissez en CSV plus tard si nécessaire.
- Pour des datasets énormes, évitez `-s/--slurp` et les agrégations inutiles; streamez avec `.[]`.

---

## Conclusion

Avec quelques filtres bien choisis, jq devient un excellent exporteur CSV : rapide, fiable, scriptable. Définissez vos colonnes, pensez à `-r` et `@csv`, gérez les cas particuliers (NULL, champs imbriqués), et vous avez tout ce qu’il faut pour automatiser vos exports.

---

## Pour aller plus loin sur linux

- [Linux : Comment changer le hostname en ligne de commande (Ubuntu/Debian)]({% post_url 2025-09-13-Comment-changer-le-hostname-en-ligne-de-commande-sur-Ubuntu-ou-Debian %})
- [Linux : Programmer une tâche avec cron (exemples utiles)]({% post_url 2025-10-11-Linux-programmer-une-tache-avec-cron %})
- [Installer et configurer Fail2ban sur un serveur Ubuntu/Debian]({% post_url 2025-09-21-Installer-et-configurer-Fail2ban-sur-Ubuntu-Debian %})
- [K8s : Comment déployer un cluster kubernetes bare-metal]({% post_url 2021-01-27-Comment-déployer-Kubernetes %})
