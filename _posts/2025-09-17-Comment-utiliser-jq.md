---
layout: article
title: "Comment manipuler du JSON en ligne de commande avec jq"
author: Pierre Chopinet
tags:
  - linux
  - jq
  - json
  - data
  - cli
  - shell
  - bash
  - outils
---

Quand une API renvoie un gros bloc de JSON dont seuls deux champs vous intéressent, pas besoin d'écrire un script Python pour les extraire : `jq` le fait directement dans le terminal. Dans ce tutoriel, nous allons voir les filtres jq dont on se sert le plus souvent, sur un petit fichier d'exemple.
<!--more-->

Dans cet article :
- Installation
- Le fichier d'exemple
- Afficher et valider du JSON
- Extraire des valeurs
- Filtrer avec select
- Trier
- Compter, additionner, grouper
- Changer la structure
- Modifier des valeurs
- Plusieurs fichiers et JSON Lines
- Passer des variables depuis le shell
- Déboguer un filtre

## Installation

jq est dans les dépôts de toutes les distributions courantes :

```bash
sudo apt install jq        # Debian, Ubuntu
sudo dnf install jq        # Fedora
sudo pacman -S jq          # Arch
brew install jq            # macOS
winget install jqlang.jq   # Windows (ou scoop install jq, ou choco install jq)
```

Si vous ne voulez rien installer, le projet publie aussi une image Docker officielle, qui lit le JSON sur l'entrée standard :

```bash
docker run --rm -i ghcr.io/jqlang/jq:latest '.users[].name' < data.json
```

Les exemples de cet article ont été testés avec jq 1.7. Pour connaître votre version : `jq --version`.

## Le fichier d'exemple

Tous les exemples utilisent ce fichier `data.json` :

```json
{
  "users": [
    {"id": 1, "name": "Alice", "active": true,  "tags": ["admin", "ops"], "score": 42.5, "created_at": "2025-09-01T12:00:00Z"},
    {"id": 2, "name": "Bob",   "active": false, "tags": ["dev"],          "score": 12.1, "created_at": "2025-08-29T10:30:00Z"},
    {"id": 3, "name": "Chloé", "active": true,  "tags": ["dev", "ops"],  "score": 31.7, "created_at": "2025-09-05T08:45:00Z"}
  ]
}
```

## Afficher et valider du JSON

Un programme jq est un filtre : il reçoit du JSON et en produit. Le plus simple est `.`, qui renvoie l'entrée telle quelle. Comme jq indente sa sortie, il suffit à rendre lisible un JSON écrit sur une seule ligne :

```bash
echo '{"id": 1, "name": "Alice", "tags": ["admin", "ops"]}' | jq .
```

Ce qui donne :

```json
{
  "id": 1,
  "name": "Alice",
  "tags": [
    "admin",
    "ops"
  ]
}
```

Dans un terminal, la sortie est en couleurs. Si on la passe à `less`, il faut forcer les couleurs avec `-C` (`jq -C . data.json | less -R`), et `-M` les désactive.

L'option `-S` trie les clés des objets. Elle sert surtout à comparer deux fichiers dont les clés ne sont pas dans le même ordre :

```bash
diff <(jq -S . v1.json) <(jq -S . v2.json)
```

jq permet aussi de vérifier qu'un fichier contient du JSON valide. Le filtre `empty` ne produit aucune sortie, seul le code de retour compte :

```bash
jq empty data.json && echo OK
```

Sur un fichier invalide, ici avec une virgule en trop, jq indique où se trouve l'erreur et sort avec un code différent de 0 (5 avec jq 1.7) :

```bash
echo '{"id": 1,}' > invalide.json
jq empty invalide.json
```

```
jq: parse error: Expected another key-value pair at line 1, column 10
```

## Extraire des valeurs

On accède à un champ avec `.nom_du_champ`, et à un élément de tableau avec son indice. Pour le nom du premier utilisateur :

```bash
jq '.users[0].name' data.json
```

```
"Alice"
```

jq affiche une chaîne JSON, avec ses guillemets. Pour récupérer le texte brut, dans une variable shell par exemple, on ajoute `-r` (*raw output*) :

```bash
jq -r '.users[0].name' data.json
```

```
Alice
```

Avec `[]` sans indice, jq parcourt tous les éléments du tableau et applique la suite du filtre à chacun :

```bash
jq -r '.users[].name' data.json
```

```
Alice
Bob
Chloé
```

Quand on ne lui donne pas de fichier, jq lit l'entrée standard, on peut donc le placer derrière un `curl`. Par exemple, pour lister les fichiers publiés sur PyPI pour la version 2.32.3 de requests :

```bash
curl -s https://pypi.org/pypi/requests/2.32.3/json | jq -r '.urls[].filename'
```

```
requests-2.32.3-py3-none-any.whl
requests-2.32.3.tar.gz
```

C'est le même principe avec `docker inspect` ou `kubectl get pods -o json`, qui produisent eux aussi du JSON.

Pour sortir une ligne de texte par élément, on construit une chaîne avec l'interpolation `\( ... )`. Le `|` fonctionne comme dans le shell : il passe chaque résultat du filtre de gauche au filtre de droite.

```bash
jq -r '.users[] | "\(.id)\t\(.name)\tactive=\(.active)"' data.json
```

On obtient des lignes séparées par des tabulations, faciles à passer à `cut` ou `awk` :

```
1	Alice	active=true
2	Bob	active=false
3	Chloé	active=true
```

Un champ absent ne provoque pas d'erreur, jq renvoie simplement `null` :

```bash
jq '.users[0].email' data.json
```

```
null
```

Dans un script, l'option `-e` permet de détecter ce cas : jq sort avec le code 1 quand la dernière valeur produite est `null` ou `false`.

```bash
jq -e '.users[0].email' data.json > /dev/null || echo "pas d'email"
```

```
pas d'email
```

Par contre, demander un champ à une chaîne, un nombre ou un tableau est une erreur : `.users[0].name.first` renvoie `Cannot index string with string "first"`, puisque `name` est une chaîne. Quand la structure des données varie d'un élément à l'autre, le suffixe `?` (`.users[0].name.first?`) fait taire l'erreur et ne produit rien.

## Filtrer avec select

`select(condition)` laisse passer les éléments pour lesquels la condition est vraie et élimine les autres. Pour le nom des utilisateurs actifs :

```bash
jq -r '.users[] | select(.active) | .name' data.json
```

```
Alice
Chloé
```

Ou celui dont l'id vaut 2 :

```bash
jq -r '.users[] | select(.id == 2) | .name' data.json
```

```
Bob
```

Sans `.name` à la fin, on récupère les objets entiers. On ajoute ici `-c` (*compact output*), qui écrit chaque objet sur une seule ligne au lieu de l'indenter :

```bash
jq -c '.users[] | select(.score >= 30)' data.json
```

```
{"id":1,"name":"Alice","active":true,"tags":["admin","ops"],"score":42.5,"created_at":"2025-09-01T12:00:00Z"}
{"id":3,"name":"Chloé","active":true,"tags":["dev","ops"],"score":31.7,"created_at":"2025-09-05T08:45:00Z"}
```

Les conditions se combinent avec `and` et `or`, et `| not` inverse une condition (`select(.active | not)` renvoie Bob). Pour les utilisateurs actifs avec un score supérieur à 40 :

```bash
jq -r '.users[] | select(.active and .score > 40) | .name' data.json
```

```
Alice
```

Pour tester le contenu d'un tableau, `any(générateur; condition)` renvoie vrai si au moins un élément vérifie la condition. Les utilisateurs qui ont le tag `dev` :

```bash
jq -r '.users[] | select(any(.tags[]; . == "dev")) | .name' data.json
```

```
Bob
Chloé
```

## Trier

`sort_by` trie un tableau selon un champ. On l'applique donc au tableau `.users` lui-même, et non à `.users[]` qui produit les éléments un par un :

```bash
jq -c '.users | sort_by(.score) | map(.name)' data.json
```

```
["Bob","Chloé","Alice"]
```

`map(f)` applique `f` à chaque élément d'un tableau et renvoie un nouveau tableau : ici, on ne garde que les noms pour que la sortie reste lisible.

Le tri est croissant, on l'inverse avec `reverse`. Les dates au format ISO 8601 se trient correctement comme de simples chaînes (tant qu'elles ont toutes le même format et le même fuseau), pas besoin de les convertir pour avoir les utilisateurs du plus récent au plus ancien :

```bash
jq -r '.users | sort_by(.created_at) | reverse | .[].name' data.json
```

```
Chloé
Alice
Bob
```

## Compter, additionner, grouper

`length` donne la taille d'un tableau et `add` additionne ses éléments. `jq '.users | length' data.json` affiche `3`, et pour la somme des scores :

```bash
jq '[.users[].score] | add' data.json
```

```
86.3
```

Les crochets autour de `.users[].score` rassemblent les scores dans un tableau, car `add` attend un tableau en entrée. La moyenne s'écrit donc :

```bash
jq '[.users[].score] | add / length' data.json
```

```
28.766666666666666
```

`min_by` et `max_by` renvoient l'élément qui a la plus petite ou la plus grande valeur pour un champ :

```bash
jq -c '.users | min_by(.score)' data.json
jq -c '.users | max_by(.score)' data.json
```

```
{"id":2,"name":"Bob","active":false,"tags":["dev"],"score":12.1,"created_at":"2025-08-29T10:30:00Z"}
{"id":1,"name":"Alice","active":true,"tags":["admin","ops"],"score":42.5,"created_at":"2025-09-01T12:00:00Z"}
```

Pour obtenir la liste des tags sans doublons, `map(.tags)` récupère les tableaux de tags, `add` les concatène en un seul tableau et `unique` trie le résultat en supprimant les doublons :

```bash
jq -c '.users | map(.tags) | add | unique' data.json
```

```
["admin","dev","ops"]
```

`unique_by` garde un seul élément par valeur de la clé, et renvoie le résultat trié par clé. Avec `.active`, `false` passe avant `true`, d'où Bob avant Alice :

```bash
jq -c '.users | unique_by(.active) | map(.name)' data.json
```

```
["Bob","Alice"]
```

Pour compter les utilisateurs actifs et inactifs, `group_by` découpe le tableau en sous-tableaux qui partagent la même valeur de clé (triés de la même façon). Il ne reste qu'à construire un objet par groupe :

```bash
jq -c '.users | group_by(.active) | map({active: .[0].active, count: length})' data.json
```

```
[{"active":false,"count":1},{"active":true,"count":2}]
```

## Changer la structure

On construit un objet avec des accolades. `{id, name, score}` est un raccourci pour `{id: .id, name: .name, score: .score}` :

```bash
jq -c '.users | map({id, name, score})' data.json
```

```
[{"id":1,"name":"Alice","score":42.5},{"id":2,"name":"Bob","score":12.1},{"id":3,"name":"Chloé","score":31.7}]
```

Les clés peuvent être renommées, et les objets imbriqués :

```bash
jq -c '.users | map({label: .name, meta: {id, active}})' data.json
```

```
[{"label":"Alice","meta":{"id":1,"active":true}},{"label":"Bob","meta":{"id":2,"active":false}},{"label":"Chloé","meta":{"id":3,"active":true}}]
```

`join` transforme un tableau de chaînes en une seule chaîne, avec le séparateur passé en paramètre. C'est souvent utile avant un export en CSV :

```bash
jq -c '.users | map({id, name, tags: (.tags | join(","))})' data.json
```

```
[{"id":1,"name":"Alice","tags":"admin,ops"},{"id":2,"name":"Bob","tags":"dev"},{"id":3,"name":"Chloé","tags":"dev,ops"}]
```

## Modifier des valeurs

L'opérateur `|=` remplace une valeur par le résultat d'un filtre appliqué à cette valeur, et `+=`, `*=`, etc. sont des raccourcis pour les opérations arithmétiques. jq renvoie le document complet, avec la modification. Pour augmenter tous les scores de 10 % (on n'affiche ensuite que le nom et le score, pour que la sortie reste lisible) :

```bash
jq -c '.users[].score *= 1.1 | .users[] | {name, score}' data.json
```

```
{"name":"Alice","score":46.75000000000001}
{"name":"Bob","score":13.31}
{"name":"Chloé","score":34.870000000000005}
```

jq calcule avec des flottants en double précision, d'où le `46.75000000000001`. Pour arrondir à deux décimales, on passe par `round` :

```bash
jq -c '.users[].score |= (. * 1.1 * 100 | round / 100) | .users[] | {name, score}' data.json
```

```
{"name":"Alice","score":46.75}
{"name":"Bob","score":13.31}
{"name":"Chloé","score":34.87}
```

Pour ne modifier que certains éléments, on peut utiliser `if ... then ... else ... end` dans un `map`, ou plus directement sélectionner le chemin à modifier. Ces deux commandes renvoient le même document, avec Bob passé en actif :

```bash
jq '.users |= map(if .name == "Bob" then .active = true else . end)' data.json
jq '(.users[] | select(.name == "Bob") | .active) = true' data.json
```

Attention, jq ne modifie jamais le fichier d'origine, il écrit le résultat sur la sortie standard. Et il ne faut surtout pas rediriger vers le fichier lu (`jq ... data.json > data.json`) : le shell vide le fichier avant que jq ne le lise, et on se retrouve avec un fichier vide. On passe par un fichier temporaire :

```bash
jq '.users[].score *= 1.1' data.json > data.tmp && mv data.tmp data.json
```

## Plusieurs fichiers et JSON Lines

Par défaut, jq applique le filtre à chaque document d'entrée, l'un après l'autre. Avec `-s` (*slurp*), il lit toutes les entrées et les range dans un seul tableau. Avec un fichier `a.json` qui contient `[1, 2]` et un fichier `b.json` qui contient `[3, 4]` :

```bash
jq -c -s '.' a.json b.json
```

```
[[1,2],[3,4]]
```

`add` concatène ensuite les deux tableaux (`flatten` donnerait ici le même résultat) :

```bash
jq -c -s 'add' a.json b.json
```

```
[1,2,3,4]
```

Le format JSON Lines, un document JSON par ligne, est courant pour les logs. jq le lit sans option particulière, puisqu'il traite les documents à la suite. Dans l'autre sens, `-c` combiné à `.[]` produit du JSON Lines à partir d'un tableau :

```bash
echo '{"items":[{"id":1},{"id":2}]}' | jq -c '.items[]'
```

```
{"id":1}
{"id":2}
```

Et `-s` reconstruit un tableau à partir d'un flux JSON Lines :

```bash
printf '%s\n' '{"id":1}' '{"id":2}' | jq -c -s '.'
```

```
[{"id":1},{"id":2}]
```

Attention, `-s` charge toutes les entrées en mémoire avant d'appliquer le filtre. Sur un gros fichier de logs, mieux vaut traiter les lignes une par une, sans *slurp*.

## Passer des variables depuis le shell

On est vite tenté de coller une variable shell dans le filtre, entre guillemets doubles. Cela fonctionne jusqu'au jour où la variable contient un guillemet ou un antislash. L'option `--arg` passe la valeur à jq sous forme de variable, sans risque :

```bash
user="Alice"

# À éviter : la valeur est insérée dans le programme jq
jq -r ".users[] | select(.name == \"$user\") | .id" data.json

# Correct :
jq -r --arg user "$user" '.users[] | select(.name == $user) | .id' data.json
```

La seconde commande affiche `1`, l'id d'Alice.

`--arg` crée toujours une chaîne. Pour un nombre ou un booléen, il faut `--argjson`, qui interprète la valeur comme du JSON. Avec `--arg`, la comparaison se fait entre un nombre et une chaîne, et pour jq un nombre est toujours plus petit qu'une chaîne : le filtre ne renvoie rien, sans la moindre erreur.

```bash
jq -c --arg threshold 30 '.users | map(select(.score > $threshold) | .name)' data.json
jq -c --argjson threshold 30 '.users | map(select(.score > $threshold) | .name)' data.json
```

```
[]
["Alice","Chloé"]
```

Avec `-n` (*null input*), jq ne lit aucune entrée. Combiné à `--arg`, c'est une façon sûre de construire un JSON à envoyer à une API, les guillemets sont échappés correctement :

```bash
jq -n --arg name 'Jean "Johnny"' --argjson score 18.5 '{name: $name, score: $score}'
```

```json
{
  "name": "Jean \"Johnny\"",
  "score": 18.5
}
```

## Déboguer un filtre

Quand un long filtre ne renvoie pas ce qu'on attend, on peut l'exécuter morceau par morceau, ou insérer `debug` à l'endroit qui pose question. `debug` laisse passer la valeur sans la modifier et l'écrit sur la sortie d'erreur. Depuis jq 1.7, `debug(msg)` permet de n'afficher qu'une partie de la valeur :

```bash
jq -r '.users[] | select(.active) | debug(.name) | .tags | join(",")' data.json
```

Dans un terminal, on obtient :

```
["DEBUG:","Alice"]
admin,ops
["DEBUG:","Chloé"]
dev,ops
```

Les lignes `DEBUG` partent sur la sortie d'erreur : `2>/dev/null` les masque sans toucher au résultat.

## Voir aussi

- [Comment transformer un JSON en CSV avec jq]({% post_url 2025-10-19-Comment-transformer-un-JSON-en-CSV-avec-jq %})
- [Ripgrep (rg) : chercher rapidement dans le code]({% post_url 2026-02-16-Chercher-dans-le-code-rapidement-avec-ripgrep %})
- [Sed : éditer des fichiers en ligne de commande avec des regex]({% post_url 2026-01-19-Sed-editer-des-fichiers-en-ligne-de-commande %})
- [Le manuel de jq 1.7](https://jqlang.org/manual/v1.7/)
- [play.jqlang.org](https://play.jqlang.org), pour tester un filtre dans le navigateur
