---
layout: article
title: "Comment transformer un JSON en CSV avec jq"
description: "Convertir du JSON en CSV avec jq en une commande : en-têtes, champs imbriqués, filtres, JSON Lines, valeurs manquantes et gros volumes en streaming."
tags:
  - linux
  - jq
  - json
  - csv
  - cli
  - data
  - outils
  - shell
  - bash
author: Pierre Chopinet
---

Pour ouvrir des données JSON dans un tableur ou les charger dans une base, il faut souvent passer par du CSV. jq sait produire un CSV correctement échappé en une seule commande, sans écrire de script : voyons comment, du cas le plus simple jusqu'aux fichiers volumineux.
<!--more-->

Dans cet article :
- Le fichier d'exemple
- Un CSV avec des colonnes choisies
- Ajouter une ligne d'en-tête
- Champs imbriqués, tableaux et valeurs manquantes
- Filtrer et trier les lignes
- Formater les valeurs
- Convertir du JSON Lines
- Le TSV, une alternative au CSV
- Les gros fichiers

Pré-requis : jq installé (les commandes ont été testées avec jq 1.7). Si vous débutez avec jq, commencez par [Comment manipuler du JSON en ligne de commande avec jq]({% post_url 2025-09-17-Comment-utiliser-jq %}).

## Le fichier d'exemple

Fichier `users.json` :

```json
[
  {"id": 1, "name": "Alice", "active": true,  "tags": ["admin", "ops"], "score": 42.5, "created_at": "2025-09-01T12:00:00Z"},
  {"id": 2, "name": "Bob",   "active": false, "tags": ["dev"],          "score": 12.1, "created_at": "2025-08-29T10:30:00Z"},
  {"id": 3, "name": "Chloé", "active": true,  "tags": ["dev", "ops"],  "score": 31.7, "created_at": "2025-09-05T08:45:00Z"}
]
```

## Un CSV avec des colonnes choisies

La méthode la plus sûre est de lister soi-même les colonnes, dans l'ordre voulu :

```bash
jq -r '.[] | [.id, .name, .active, .score] | @csv' users.json
```

Ce qui donne :

```
1,"Alice",true,42.5
2,"Bob",false,12.1
3,"Chloé",true,31.7
```

`.[]` parcourt les objets du tableau, `[.id, .name, .active, .score]` construit pour chacun un tableau avec les valeurs des colonnes, et `@csv` transforme ce tableau en une ligne CSV. Les chaînes sont mises entre guillemets, les nombres et les booléens non.

Il ne faut pas oublier l'option `-r`. Sans elle, jq affiche chaque ligne comme une chaîne JSON, entre guillemets et avec les guillemets intérieurs échappés :

```
"1,\"Alice\",true,42.5"
```

L'intérêt de `@csv` par rapport à une concaténation de chaînes faite à la main, c'est l'échappement. Une valeur qui contient une virgule ou des guillemets reste correcte :

```bash
echo '[{"id": 1, "name": "Dupont, Jean", "note": "Il a dit \"oui\""}]' | jq -r '.[] | [.id, .name, .note] | @csv'
```

```
1,"Dupont, Jean","Il a dit ""oui"""
```

Les guillemets sont doublés, comme le prévoit le format CSV.

## Ajouter une ligne d'en-tête

Le plus simple est d'écrire l'en-tête avec `printf`, puis d'ajouter les lignes produites par jq. Les accolades regroupent les deux commandes pour rediriger leur sortie vers un seul fichier :

```bash
{
  printf 'id,name,active,score\n'
  jq -r '.[] | [.id, .name, .active, .score] | @csv' users.json
} > users.csv
```

Le fichier `users.csv` contient alors :

```
id,name,active,score
1,"Alice",true,42.5
2,"Bob",false,12.1
3,"Chloé",true,31.7
```

Pour renommer les colonnes, il suffit de mettre d'autres noms dans le `printf` (`user_id,full_name,is_active,score` par exemple).

On peut aussi construire l'en-tête à partir des clés du premier objet. `keys_unsorted` renvoie les clés dans l'ordre où elles apparaissent dans l'objet (`keys` les trierait par ordre alphabétique), et `.[$keys[]]` récupère les valeurs dans ce même ordre :

```bash
jq -r '(.[0] | keys_unsorted) as $keys
  | $keys, (.[] | [.[$keys[]] | if type == "array" then join(";") else . end])
  | @csv' users.json
```

```
"id","name","active","tags","score","created_at"
1,"Alice",true,"admin;ops",42.5,"2025-09-01T12:00:00Z"
2,"Bob",false,"dev",12.1,"2025-08-29T10:30:00Z"
3,"Chloé",true,"dev;ops",31.7,"2025-09-05T08:45:00Z"
```

Le `if type == "array"` est nécessaire à cause de la colonne `tags`. `@csv` n'accepte que des chaînes, des nombres, des booléens et `null` : sans cette conversion, jq affiche l'en-tête puis s'arrête dès la ligne d'Alice avec l'erreur `array (["admin","o...) is not valid in a csv row`.

Attention, les colonnes sont celles du premier objet. Une clé qui n'apparaît que dans les objets suivants est ignorée, sans avertissement. Si les clés varient d'un objet à l'autre, on peut prendre l'union de toutes les clés, au prix d'un ordre alphabétique des colonnes. Ici, seul Bob a un email :

```bash
echo '[{"id": 1, "name": "Alice"}, {"id": 2, "name": "Bob", "email": "bob@example.com"}]' \
  | jq -r '(map(keys) | add | unique) as $keys | $keys, (.[] | [.[$keys[]]]) | @csv'
```

```
"email","id","name"
,1,"Alice"
"bob@example.com",2,"Bob"
```

Pour un export qui doit rester stable dans le temps, la liste explicite de colonnes reste préférable.

## Champs imbriqués, tableaux et valeurs manquantes

Pour un champ imbriqué, on écrit simplement le chemin complet. Un champ absent vaut `null`, que `@csv` écrit comme un champ vide :

```bash
echo '[{"id": 1, "profile": {"city": "Lyon"}}, {"id": 2}]' | jq -r '.[] | [.id, .profile.city] | @csv'
```

```
1,"Lyon"
2,
```

Pour mettre une autre valeur par défaut, on utilise l'opérateur `//`, qui renvoie la valeur de droite quand celle de gauche est `null` ou `false`. Avec `(.profile.city // "inconnue")`, la deuxième ligne devient `2,"inconnue"`. De la même façon, `(.score // 0)` remplace un score manquant par 0. Attention aux colonnes booléennes : `//` remplace aussi les `false`, et avec `[.id, (.active // "")]` la ligne de Bob devient `2,""` au lieu de `2,false`.

Un tableau doit être converti en chaîne avant de passer dans `@csv`. `join` assemble ses éléments avec le séparateur de votre choix :

```bash
jq -r '.[] | [.id, .name, (.tags | join(";")), .score] | @csv' users.json
```

```
1,"Alice","admin;ops",42.5
2,"Bob","dev",12.1
3,"Chloé","dev;ops",31.7
```

Évitez par contre `tostring` sur les nombres et les booléens. `@csv` met entre guillemets toutes les chaînes, donc `[(.id | tostring), .name, (.active | tostring)]` donne `"1","Alice","true"` au lieu de `1,"Alice",true`. Ce n'est utile que si l'outil qui lit le CSV attend justement ces guillemets.

## Filtrer et trier les lignes

Comme on construit chaque ligne avec un filtre jq, on peut filtrer et trier avant l'export. Pour les utilisateurs actifs uniquement, du meilleur score au moins bon, avec trois colonnes :

```bash
jq -r 'map(select(.active))
  | sort_by(.score) | reverse
  | .[] | [.id, .name, .score]
  | @csv' users.json
```

```
1,"Alice",42.5
3,"Chloé",31.7
```

`map(select(.active))` garde les objets dont `active` vaut `true`, `sort_by(.score) | reverse` les trie par score décroissant, puis on retrouve la même construction que précédemment.

## Formater les valeurs

`round` arrondit un nombre à l'entier le plus proche :

```bash
jq -r '.[] | [.id, .name, (.score | round)] | @csv' users.json
```

```
1,"Alice",43
2,"Bob",12
3,"Chloé",32
```

Pour garder deux décimales, on multiplie avant d'arrondir : `(.score * 100 | round / 100)`.

Quand une API renvoie les nombres sous forme de chaînes (`"score": "42.5"`), `@csv` les met entre guillemets : `1,"42.5"`. `(.score | tonumber)` les reconvertit, et on obtient `1,42.5`.

Pour ne garder que la date d'un horodatage ISO 8601, il suffit de couper la chaîne au `T` :

```bash
jq -r '.[] | [.id, (.created_at | split("T")[0])] | @csv' users.json
```

```
1,"2025-09-01"
2,"2025-08-29"
3,"2025-09-05"
```

## Convertir du JSON Lines

Le format JSON Lines (ou NDJSON) contient un objet JSON par ligne, sans tableau autour. Fichier `users.jsonl` :

```
{"id":1,"name":"Alice","active":true,"score":42.5}
{"id":2,"name":"Bob","active":false,"score":12.1}
{"id":3,"name":"Chloé","active":true,"score":31.7}
```

jq traite chaque ligne comme une entrée séparée, il n'y a donc plus de `.[]` dans le filtre :

```bash
{
  printf 'id,name,active,score\n'
  jq -r '[.id, .name, .active, .score] | @csv' users.jsonl
} > users.csv
```

Pour construire l'en-tête à partir du premier objet, on lance jq avec `-n` pour qu'il ne lise pas les entrées tout seul. `input` lit alors le premier objet, et `inputs` les suivants, un par un :

```bash
jq -n -r 'input as $first
  | ($first | keys_unsorted) as $keys
  | $keys, (($first, inputs) | [.[$keys[]]])
  | @csv' users.jsonl
```

```
"id","name","active","score"
1,"Alice",true,42.5
2,"Bob",false,12.1
3,"Chloé",true,31.7
```

Le fichier n'est jamais chargé en entier en mémoire, ce qui compte sur un gros fichier (voir la dernière partie).

## Le TSV, une alternative au CSV

`@tsv` fonctionne comme `@csv`, mais sépare les colonnes par des tabulations et ne met pas de guillemets. C'est pratique quand les valeurs contiennent beaucoup de virgules. Les tabulations, retours à la ligne et antislashs présents dans les valeurs sont échappés en `\t`, `\n` et `\\` : chaque objet tient donc toujours sur une ligne.

```bash
jq -r '.[] | [.id, .name, .active, .score] | @tsv' users.json > users.tsv
```

Le TSV a aussi l'avantage de s'afficher proprement dans un terminal avec `column` :

```bash
column -t -s $'\t' users.tsv
```

```
1  Alice  true   42.5
2  Bob    false  12.1
3  Chloé  true   31.7
```

## Les gros fichiers

Sur un gros tableau JSON, jq lit tout le document en mémoire avant d'appliquer le filtre, et le fait d'écrire `.[]` plutôt que `map(...)` n'y change presque rien. Pour donner un ordre de grandeur, sur un tableau de 500 000 objets semblables à ceux de l'exemple (un fichier de 63 Mo), la commande `jq -r '.[] | [.id, .name, .score] | @csv'` a utilisé environ 600 Mo de mémoire avec jq 1.7.

Si la mémoire pose problème, l'option `--stream` fait lire le document au fil de l'eau, sous forme d'événements (un chemin et une valeur). `1 | truncate_stream(inputs)` retire l'indice du tableau de ces chemins, et `fromstream` reconstruit chaque objet :

```bash
jq -n -r --stream 'fromstream(1 | truncate_stream(inputs))
  | [.id, .name, .score]
  | @csv' huge.json > out.csv
```

Sur le même fichier, la mémoire utilisée tombe à une dizaine de Mo, mais la conversion a été environ trois fois plus lente (16 secondes au lieu de 5 sur la machine de test).

Le JSON Lines n'a pas ce problème : jq ne garde en mémoire qu'une ligne à la fois. Les mêmes données au format JSON Lines ont été converties en moins de 2 secondes, avec une dizaine de Mo de mémoire. Si vous avez le choix du format, c'est donc lui qu'il faut privilégier pour les gros volumes. Évitez par contre l'option `-s` (*slurp*) sur ces fichiers, puisqu'elle charge toutes les lignes dans un tableau.

Pour suivre l'avancement d'une longue conversion, on peut placer `pv` avant jq. Comme `pv` connaît la taille du fichier, il affiche une barre de progression avec un pourcentage :

```bash
pv huge.jsonl | jq -r '[.id, .name, .score] | @csv' > out.csv
```

## Voir aussi

- [Comment manipuler du JSON en ligne de commande avec jq]({% post_url 2025-09-17-Comment-utiliser-jq %})
- [Python : Comment sauvegarder et charger un dataframe Pandas avec Excel (ou du csv)]({% post_url 2023-12-28-Comment-sauvegarder-un-dataframe-pandas %})
- [Le manuel de jq 1.7](https://jqlang.org/manual/v1.7/), en particulier la partie *Format strings and escaping*
- [La RFC 4180](https://www.rfc-editor.org/rfc/rfc4180), qui décrit le format CSV
