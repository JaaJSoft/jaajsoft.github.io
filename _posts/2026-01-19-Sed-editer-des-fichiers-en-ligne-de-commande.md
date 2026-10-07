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

`sed` (*stream editor*) lit un texte ligne par ligne et lui applique des commandes d'édition : remplacer un mot, supprimer ou insérer des lignes. C'est l'outil classique pour modifier un fichier de configuration depuis un script, ou pour retoucher la sortie d'une autre commande dans un pipe.
<!--more-->

Dans cet article :
- Installation
- La syntaxe de base
- Remplacer du texte
- Supprimer, insérer ou remplacer des lignes
- Afficher seulement certaines lignes
- Les expressions régulières
- Modifier les fichiers en place
- Nettoyer un fichier
- Travailler sur plusieurs lignes
- GNU sed et BSD sed
- Quand passer à un autre outil

## Installation

`sed` est présent sur toutes les distributions Linux, dans sa version GNU. Sur macOS, la version installée est celle de BSD, qui diffère sur quelques points (voir plus bas). On peut y installer GNU sed avec Homebrew (`brew install gnu-sed`), il est alors disponible sous le nom `gsed`. Sous Windows, le plus simple est de passer par WSL ou Git Bash.

Les exemples de cet article ont été testés avec GNU sed 4.9 :

```bash
sed --version | head -1
```

```
sed (GNU sed) 4.9
```

## La syntaxe de base

```bash
sed 'script' fichier.txt                # Affiche le résultat, le fichier n'est pas modifié
sed -e 'commande1' -e 'commande2' fichier.txt
commande | sed 'script'                 # Travaille sur l'entrée standard
```

Le script est une suite de commandes, chacune pouvant être précédée d'une adresse qui indique les lignes concernées : un numéro de ligne, un intervalle ou une expression régulière. Sans adresse, la commande s'applique à toutes les lignes.

Par défaut, sed écrit le résultat sur la sortie standard et ne touche pas au fichier. On peut donc essayer une commande autant de fois que nécessaire avant de l'appliquer pour de bon avec `-i` (voir plus bas).

Pour les exemples, on utilise ce fichier `app.conf` :

```
# Configuration de l'application
host = localhost
port = 8080
debug = true

log_level = info
api_url = http://example.com/api
```

## Remplacer du texte

La commande la plus utilisée est `s/motif/remplacement/` :

```bash
sed 's/localhost/127.0.0.1/' app.conf
```

Ce qui donne :

```
# Configuration de l'application
host = 127.0.0.1
port = 8080
debug = true

log_level = info
api_url = http://example.com/api
```

Sans option, `s` ne remplace que la première occurrence de chaque ligne. Les *flags* placés après le dernier `/` changent ce comportement :

```bash
echo "foo foo foo" | sed 's/foo/bar/'     # bar foo foo
echo "foo foo foo" | sed 's/foo/bar/g'    # bar bar bar (toutes les occurrences)
echo "foo foo foo" | sed 's/foo/bar/2'    # foo bar foo (seulement la 2e)
echo "Foo FOO foo" | sed 's/foo/bar/gI'   # bar bar bar (sans tenir compte de la casse)
```

Le flag `I` (ou `i`) est une extension GNU.

Le `/` n'est pas obligatoire comme séparateur : n'importe quel caractère qui suit le `s` fait l'affaire. C'est bien pratique avec des chemins, pour éviter une forêt d'antislashs :

```bash
sed 's/\/old\/path/\/new\/path/g' fichier.txt   # Illisible
sed 's|/old/path|/new/path|g' fichier.txt       # Même chose
```

Dans le remplacement, `&` représente tout le texte trouvé :

```bash
echo "port = 8080" | sed 's/[0-9]\+/"&"/'
```

```
port = "8080"
```

GNU sed sait aussi changer la casse dans le remplacement : `\U` passe la suite en majuscules, `\L` en minuscules, et `\u` ne met en majuscule que le caractère suivant. Pour mettre une majuscule au début de chaque mot :

```bash
echo "bonjour le monde" | sed 's/\b./\u&/g'
```

```
Bonjour Le Monde
```

Ces séquences gèrent les lettres accentuées tant que la locale est en UTF-8 (`élise` devient `Élise`). On voit parfois conseiller `tr '[:lower:]' '[:upper:]'` à la place, mais la version GNU de `tr` travaille octet par octet et laisse les lettres accentuées en minuscules.

Pour enchaîner plusieurs commandes, on répète `-e` ou on les sépare par des `;` :

```bash
sed -e 's/foo/bar/g' -e 's/hello/world/g' fichier.txt
sed 's/foo/bar/g; s/hello/world/g' fichier.txt
```

Au-delà de quelques commandes, mieux vaut les écrire dans un fichier, une par ligne, et le passer avec `-f` :

```bash
sed -f script.sed fichier.txt
```

## Supprimer, insérer ou remplacer des lignes

`d` supprime les lignes désignées par l'adresse :

```bash
sed '3d' app.conf          # La ligne 3
sed '2,4d' app.conf        # Les lignes 2 à 4
sed '$d' app.conf          # La dernière ligne
sed '/^#/d' app.conf       # Les lignes qui commencent par #
sed '/debug/d' app.conf    # Les lignes qui contiennent "debug"
```

`i` insère du texte avant la ligne, `a` l'ajoute après, et `c` remplace la ligne entière. L'adresse peut là aussi être un numéro ou une regex. Pour ajouter un paramètre après la ligne du port :

```bash
sed '/^port/a timeout = 30' app.conf
```

```
# Configuration de l'application
host = localhost
port = 8080
timeout = 30
debug = true

log_level = info
api_url = http://example.com/api
```

Et pour remplacer la ligne `debug` :

```bash
sed '/^debug/c debug = false' app.conf
```

Écrire le texte sur la même ligne que la commande est une extension GNU. La forme portable met un `\` après la commande et le texte sur la ligne suivante :

```bash
sed '/^port/a\
timeout = 30' app.conf
```

Attention aussi, tout ce qui suit `a`, `i` ou `c` fait partie du texte : on ne peut pas enchaîner une autre commande avec un `;` derrière, il faut un autre `-e`.

Pour insérer le contenu d'un fichier, on utilise `r`. La commande ajoute le fichier après la ligne visée, sans supprimer cette ligne. Pour vraiment remplacer la ligne 5 par le contenu de `insert.txt`, on combine `r` et `d` :

```bash
seq 6 > six.txt
printf 'inséré 1\ninséré 2\n' > insert.txt
sed -e '5{r insert.txt' -e 'd}' six.txt
```

```
1
2
3
4
inséré 1
inséré 2
6
```

Les deux `-e` ne sont pas là par hasard : le nom de fichier de `r` va jusqu'à la fin de la ligne, un `;` ou un `}` sur la même ligne ferait partie du nom. Et si le fichier n'existe pas, sed n'affiche aucune erreur : la ligne est simplement supprimée.

## Afficher seulement certaines lignes

Par défaut, sed affiche toutes les lignes. L'option `-n` désactive cet affichage, et la commande `p` affiche explicitement les lignes voulues :

```bash
sed -n '5p' fichier.txt          # La ligne 5
sed -n '10,20p' fichier.txt      # Les lignes 10 à 20
sed -n '$p' fichier.txt          # La dernière ligne
sed -n '/error/p' fichier.txt    # Les lignes qui contiennent "error", comme grep
```

Combiné à `s`, le flag `p` n'affiche que les lignes où un remplacement a eu lieu, ce qui permet de vérifier ce qu'une substitution va toucher :

```bash
sed -n 's/localhost/127.0.0.1/p' app.conf
```

```
host = 127.0.0.1
```

Une adresse peut aussi être un intervalle entre deux regex. Avec un fichier `bloc.txt` qui contient `avant`, `START`, `ligne a`, `ligne b`, `END` et `après` (une valeur par ligne) :

```bash
sed -n '/START/,/END/p' bloc.txt
```

```
START
ligne a
ligne b
END
```

Le même intervalle avec `d` à la place de `-n` et `p` supprime le bloc, bornes comprises.

Enfin, la commande `=` affiche le numéro de chaque ligne. Combinée à un second sed qui recolle le numéro et la ligne, elle numérote un fichier :

```bash
sed = app.conf | sed 'N; s/\n/\t/'
```

`nl app.conf` fait presque la même chose, mais ne numérote pas les lignes vides par défaut. Il faut `nl -b a` pour qu'il les numérote toutes.

## Les expressions régulières

Par défaut, sed utilise les expressions régulières basiques (BRE). Les caractères `.`, `*`, `^`, `$` et `[ ]` y ont leur sens habituel, mais `+`, `?`, `|`, `( )` et `{ }` sont des caractères normaux : pour leur donner un sens spécial, il faut les précéder d'un antislash.

```bash
echo "commande 42 du 12/03/2025" | sed 's/[0-9]\+/NUM/g'
```

```
commande NUM du NUM/NUM/NUM
```

Sans l'antislash, `s/[0-9]+/NUM/g` cherche un chiffre suivi d'un vrai `+` et ne remplace rien. Notez que `\+`, `\?` et `\|` sont des extensions GNU.

Avec l'option `-E`, sed passe aux expressions régulières étendues (ERE), où ces caractères sont spéciaux sans antislash. C'est plus lisible dès que l'expression contient des groupes. `-r` est l'ancien nom de cette option chez GNU, mais `-E` est accepté aussi par BSD sed et fait désormais partie de la norme POSIX.

Les groupes entre parenthèses se réutilisent dans le remplacement avec `\1`, `\2`, etc. Pour passer des dates du format `JJ/MM/AAAA` au format `AAAA-MM-JJ` :

```bash
echo "Né le 12/03/2025, inscrit le 01/09/2025" | sed -E 's|([0-9]{2})/([0-9]{2})/([0-9]{4})|\3-\2-\1|g'
```

```
Né le 2025-03-12, inscrit le 2025-09-01
```

GNU sed propose aussi quelques raccourcis : `\w` pour un caractère de mot (lettre, chiffre ou `_`), `\s` pour un espace ou une tabulation, `\b` pour une limite de mot. Pour inverser deux mots :

```bash
echo "Jean Dupont" | sed -E 's/(\w+) (\w+)/\2 \1/'
```

```
Dupont Jean
```

Attention au point, qui remplace n'importe quel caractère. Pour remplacer la version `1.5`, il faut l'échapper, sinon `105` est remplacé aussi :

```bash
echo "version 1.5 et 105" | sed 's/1.5/2.0/g'     # version 2.0 et 2.0
echo "version 1.5 et 105" | sed 's/1\.5/2.0/g'    # version 2.0 et 105
```

Si vous mettez au point vos regex sur un site comme regex101, gardez en tête que ses moteurs (PCRE, JavaScript...) ne sont pas ceux de sed. Par exemple, `\d` ne désigne pas un chiffre dans sed : `sed -E 's/\d+/X/g'` ne renvoie aucune erreur et ne remplace rien. Il faut écrire `[0-9]` ou `[[:digit:]]`. Les *lookarounds* (`(?=...)`) ne sont pas disponibles non plus.

Pour extraire une valeur, on combine `-n`, un groupe qui capture la valeur et le flag `p`. Il faut alors se méfier du `.*` en début de motif, qui est gourmand : il avale le plus de caractères possible. Avec cette tentative d'extraction d'une adresse email :

```bash
echo "Contact : alice.martin@example.com" | sed -nE 's/.*([a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}).*/\1/p'
```

On obtient :

```
n@example.com
```

Le `.*` a pris tout ce qu'il pouvait, en ne laissant qu'un caractère au groupe. Pour ce genre d'extraction, `grep -o` est plus adapté : il affiche seulement la partie qui correspond au motif, et trouve aussi plusieurs occurrences sur une même ligne.

```bash
echo "Contact : alice.martin@example.com" | grep -oE '[[:alnum:]._%+-]+@[[:alnum:].-]+\.[[:alpha:]]{2,}'
```

```
alice.martin@example.com
```

## Modifier les fichiers en place

L'option `-i` écrit le résultat dans le fichier au lieu de l'afficher. Avec un suffixe collé à l'option, sed garde une copie du fichier d'origine :

```bash
sed -i 's/8080/9090/' app.conf          # Modifie app.conf directement
sed -i.bak 's/8080/9090/' app.conf      # Modifie app.conf et garde l'original dans app.conf.bak
```

Comme il n'y a pas de retour en arrière possible sans sauvegarde, je vous conseille de toujours lancer la commande une première fois sans `-i` pour vérifier le résultat.

Attention à deux pièges, décrits dans le manuel de GNU sed. Le suffixe de `-i` étant collé à l'option, `sed -iE '...'` ne veut pas dire `-i -E` : sed crée une sauvegarde nommée `app.confE`. Écrivez plutôt `sed -E -i` ou `sed -Ei`. Et `-n` avec `-i` vide le fichier si le script ne contient pas de `p` : `sed -ni 's/8080/9090/' app.conf` laisse un fichier vide.

Pour modifier plusieurs fichiers d'un coup, on combine sed avec `find`. Ici, on remplace `http://` par `https://` dans tous les fichiers `.txt` de l'arborescence :

```bash
find . -name "*.txt" -exec sed -i 's|http://|https://|g' {} +
```

Avec [ripgrep]({% post_url 2026-02-16-Chercher-dans-le-code-rapidement-avec-ripgrep %}), `rg -l motif` liste les fichiers qui contiennent un motif, et `xargs` les passe à sed : `rg -l 'old_function' | xargs sed -i 's/old_function/new_function/g'`.

## Nettoyer un fichier

Supprimer les lignes vides, et celles qui ne contiennent que des espaces ou des tabulations :

```bash
sed '/^$/d' fichier.txt
sed '/^[[:space:]]*$/d' fichier.txt
```

Supprimer les espaces et tabulations en début et en fin de ligne :

```bash
sed 's/^[[:space:]]*//; s/[[:space:]]*$//' fichier.txt
```

On voit souvent `[ \t]` à la place de `[[:space:]]`. Cela fonctionne avec GNU sed, mais reconnaître `\t` dans des crochets est une extension GNU : `[[:space:]]` ou `[[:blank:]]` sont portables.

Supprimer les commentaires, en ligne entière ou en fin de ligne :

```bash
sed '/^#/d' fichier.txt
sed 's/#.*$//' fichier.txt
```

La seconde commande est à manier avec précaution : elle coupe aussi les `#` qui ne sont pas des commentaires. Sur une ligne `color = #ff0000   # rouge`, il ne reste que `color =`.

Commenter, puis décommenter, la ligne `debug` d'un fichier de configuration :

```bash
sed -i '/^debug/s/^/#/' app.conf        # debug = true devient #debug = true
sed -i 's/^#\(debug\)/\1/' app.conf     # et inversement
```

La première commande montre qu'on peut mettre une adresse devant `s` : le remplacement ne s'applique qu'aux lignes qui correspondent à `/^debug/`.

Enfin, cette commande supprime les lignes identiques consécutives :

```bash
sed '$!N; /^\(.*\)\n\1$/!P; D' fichier.txt
```

Elle fait exactement la même chose que `uniq fichier.txt`, bien plus court à écrire. Pour supprimer tous les doublons, consécutifs ou non, il faut trier avant : `sort fichier.txt | uniq`, en sachant que l'ordre des lignes est alors perdu.

## Travailler sur plusieurs lignes

sed traite les lignes une par une, ce qui complique les remplacements à cheval sur plusieurs lignes. La commande `N` ajoute la ligne suivante à la ligne en cours, séparée par un `\n`. Pour joindre toutes les lignes d'un fichier avec des espaces :

```bash
printf 'un\ndeux\ntrois\n' | sed ':a;N;$!ba;s/\n/ /g'
```

```
un deux trois
```

`:a` définit une étiquette, `N` ajoute la ligne suivante, `$!ba` revient à l'étiquette tant qu'on n'est pas sur la dernière ligne, puis `s/\n/ /g` remplace tous les retours à la ligne accumulés. `paste -sd ' ' fichier.txt` donne le même résultat. `tr '\n' ' '` remplace aussi le dernier retour à la ligne, et la sortie se termine alors par une espace au lieu d'un saut de ligne.

À l'inverse, `sed G` ajoute une ligne vide après chaque ligne, ce qui double l'interligne d'un fichier.

## GNU sed et BSD sed

Le sed de macOS est la version BSD. La différence la plus gênante concerne `-i` : BSD sed exige un suffixe de sauvegarde, éventuellement vide.

```bash
sed -i '' 's/foo/bar/g' fichier.txt     # BSD sed (macOS)
sed -i 's/foo/bar/g' fichier.txt        # GNU sed (Linux)
```

La forme BSD ne fonctionne pas avec GNU sed, qui prend `''` pour le script et `s/foo/bar/g` pour un nom de fichier (`sed: can't read s/foo/bar/g: No such file or directory`). Pour un script qui doit tourner sur les deux, le plus simple est d'utiliser un suffixe collé (`-i.bak`), puis de supprimer la sauvegarde.

Plusieurs fonctionnalités utilisées dans cet article sont des extensions GNU, d'après le manuel de GNU sed : `\+`, `\?` et `\|` dans les regex basiques, `\w`, `\s` et `\b`, les séquences `\U`, `\L` et `\u`, le flag `I`, le texte sur la même ligne que `a`, `i` et `c`, ou encore `\t` dans des crochets. Pour vérifier qu'un script s'en passe, GNU sed propose l'option `--posix`, qui désactive toutes ces extensions. Sur un Mac, le plus simple reste d'installer GNU sed et d'utiliser `gsed`.

## Quand passer à un autre outil

sed est fait pour des transformations ligne par ligne. Dès qu'il faut raisonner en colonnes ou faire des calculs, `awk` est plus adapté. Pour du JSON, `jq` comprend la structure du document, là où une regex finira par casser sur un cas particulier, et c'est pareil pour le HTML ou le XML avec un vrai parseur. Enfin, pour des regex complexes (*lookarounds*, Unicode), `perl` ou un petit script Python seront plus confortables.

## Voir aussi

- [Ripgrep (rg) : chercher rapidement dans le code]({% post_url 2026-02-16-Chercher-dans-le-code-rapidement-avec-ripgrep %})
- [Comment manipuler du JSON en ligne de commande avec jq]({% post_url 2025-09-17-Comment-utiliser-jq %})
- [Le manuel de GNU sed](https://www.gnu.org/software/sed/manual/), aussi disponible en local avec `info sed`
- [Useful one-line scripts for sed](http://www.pement.org/sed/sed1line.txt), la liste de *one-liners* d'Eric Pement
