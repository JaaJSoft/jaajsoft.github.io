# CLAUDE.md

Blog technique en français (blog.jaaj.dev), Jekyll + thème TeXt. Les articles vivent dans `_posts/`.

## Règles éditoriales des articles

### Style et typographie
- Jamais de tiret cadratin ni demi-cadratin : utiliser `-`, `:` ou des parenthèses. Pas de trait d'union insécable (U+2011), apostrophes droites (`'`).
- Pas de titres de sections numérotés (`## Méthode 1 :`, `### 3)`) : titres descriptifs uniquement.
- Pas d'emojis dans le contenu ni le code : écrire "À faire" / "À éviter", et en commentaire `// À éviter :` / `// Correct :`.
- Pas de blocs "insight" ou d'encadrés de style IA dans les articles.
- Typographie française : espace avant `:` et `;` en prose. Attention : les articles existants utilisent souvent une espace insécable (U+00A0) avant `:`, à reproduire dans les Edit pour que le old_string matche.
- Ton naturel francophone, tutoiement absent, style de l'auteur conservé.

### Structure d'un article
- Frontmatter : `layout: article`, `title`, `tags`, `author: Pierre Chopinet`.
- Balise `<!--more-->` obligatoire après le paragraphe d'introduction (extrait).
- Sommaire "Dans cet article :" en tête : il doit lister toutes les sections réelles.
- Titres de sections en `##` (jamais de `#` dans le corps).
- Les libellés des liens croisés doivent reprendre le titre exact de l'article cible.

### Liens internes
- `{% post_url ... %}` uniquement vers un fichier existant de `_posts/` : une référence vers un article inexistant casse le build Jekyll.
- Vérification rapide : comparer `grep -rho 'post_url [^ %]*' _posts` avec `ls _posts`.

### Exactitude technique
- Exécuter ou compiler chaque snippet avant publication ; les sorties console affichées doivent être les vraies (piège récurrent : l'ordre d'itération d'une HashMap n'est pas garanti, le noter à la première sortie).
- Épingler ou dater les versions des bibliothèques sensibles ("testé avec X v1.2") pour éviter qu'un article devienne silencieusement faux.
- Vérifier les affirmations contre la doc officielle, pas de mémoire.

### Articles obsolètes
- Ne pas réécrire ni supprimer : ajouter un bandeau blockquote en tête de contenu, ex. `> **Note (2026) :** cet article date de X et n'est plus réalisable en l'état... Il est conservé à titre historique.` Corriger le code manifestement cassé, mais ne pas moderniser les API mortes.

## Notes techniques
- Ce fichier est listé dans `exclude:` de `_config.yml` : Jekyll ne doit JAMAIS le traiter (les exemples `post_url` ci-dessus casseraient le build). Ne pas le retirer de la liste.
- Fichiers en UTF-8 ; le `grep -P` de Git Bash échoue sur les classes Unicode, utiliser `perl -CSD` pour chercher/remplacer des caractères spéciaux.
- Le dossier `docs/` appartient au thème TeXt (démo/documentation upstream) : ne pas y toucher lors des passes sur les articles.
