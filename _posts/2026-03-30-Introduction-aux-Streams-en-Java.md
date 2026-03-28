---
layout: article
title: "Introduction aux Streams en Java"
author: Pierre Chopinet
tags:
  - java
  - streams
  - collections
  - functional
---

L'API Stream, introduite en Java 8, permet de traiter des collections de données de manière déclarative : filtrer, transformer, trier et agréger en enchaînant des opérations, sans écrire de boucles. Elle s'appuie sur un style fonctionnel qui rend le code plus concis et plus lisible.
<!--more-->

Dans cet article, vous découvrirez :
- Ce qu'est un Stream et en quoi il diffère d'une collection
- Comment créer des Streams depuis différentes sources
- Les opérations intermédiaires : `filter`, `map`, `flatMap`, `sorted`, `distinct`, `limit`, `skip`
- Les opérations terminales : `collect`, `forEach`, `reduce`, `count`, `findFirst`, `anyMatch`
- Les Collectors les plus utiles : `toList`, `toSet`, `toMap`, `joining`
- Les Streams de primitifs (`IntStream`, `DoubleStream`, `LongStream`)
- Les Streams parallèles et quand les utiliser
- Les pièges courants à éviter

Pré-requis : Java 8+ (Java 16+ pour `Stream.toList()`).

---

## Qu'est-ce qu'un Stream ?

Un Stream est un **pipeline de traitement de données**. Contrairement à une collection qui stocke des éléments, un Stream ne stocke rien : il décrit une séquence d'opérations à appliquer sur une source de données.

```java
List<String> noms = List.of("Alice", "Bob", "Charlie", "David", "Eve");

List<String> resultat = noms.stream()
    .filter(nom -> nom.length() > 3)
    .map(String::toUpperCase)
    .sorted()
    .toList();

System.out.println(resultat); // [ALICE, CHARLIE, DAVID]
```

Ce code filtre les noms de plus de 3 caractères, les met en majuscules et les trie, le tout sans écrire une seule boucle.

### Stream vs Collection

| Caractéristique | Collection                  | Stream                                 |
|-----------------|-----------------------------|----------------------------------------|
| Stockage        | Oui, conserve les éléments  | Non, traite les éléments à la volée    |
| Réutilisation   | Utilisable plusieurs fois   | Usage unique (consommé après terminal) |
| Évaluation      | Immédiate (eager)           | Paresseuse (lazy)                      |
| Modification    | Mutable (add, remove)       | Immuable (ne modifie pas la source)    |
| Taille          | Finie                       | Peut être infinie                      |

### L'évaluation paresseuse (lazy)

Les opérations intermédiaires ne s'exécutent pas immédiatement. Elles sont empilées et ne sont déclenchées que lorsqu'une opération terminale est appelée :

```java
List<String> noms = List.of("Alice", "Bob", "Charlie");

// Rien ne se passe ici : aucune opération terminale
Stream<String> stream = noms.stream()
    .filter(nom -> {
        System.out.println("Filtrage de : " + nom);
        return nom.length() > 3;
    });

System.out.println("Avant l'opération terminale");

// C'est le toList() qui déclenche tout le pipeline
List<String> resultat = stream.toList();
```

**Sortie :**
```
Avant l'opération terminale
Filtrage de : Alice
Filtrage de : Bob
Filtrage de : Charlie
```

Les messages de filtrage apparaissent **après** "Avant l'opération terminale", car le pipeline ne s'exécute qu'au moment du `toList()`.

---

## Créer un Stream

### Depuis une collection

La méthode `stream()` est disponible sur toute collection :

```java
List<String> liste = List.of("a", "b", "c");
Stream<String> stream = liste.stream();

Set<Integer> ensemble = Set.of(1, 2, 3);
Stream<Integer> stream2 = ensemble.stream();
```

### Depuis un tableau

```java
String[] tableau = {"a", "b", "c"};
Stream<String> stream = Arrays.stream(tableau);
```

### Avec Stream.of()

Pour créer un Stream à partir de valeurs directes :

```java
Stream<String> stream = Stream.of("Alice", "Bob", "Charlie");
```

### Avec Stream.generate() et Stream.iterate()

Pour des Streams potentiellement infinis :

```java
// Génère une séquence infinie de nombres aléatoires
Stream<Double> aleatoires = Stream.generate(Math::random);

// Génère 0, 2, 4, 6, 8, ...
Stream<Integer> pairs = Stream.iterate(0, n -> n + 2);

// Avec limite (Java 9+) : 0, 2, 4, ..., 18
Stream<Integer> pairsLimites = Stream.iterate(0, n -> n < 20, n -> n + 2);
```

> Les Streams infinis doivent être limités avec `limit()` ou un prédicat pour éviter une boucle infinie.

### Depuis un fichier

```java
try (Stream<String> lignes = Files.lines(Path.of("fichier.txt"))) {
    lignes.filter(l -> !l.isBlank())
        .forEach(System.out::println);
}
```

### Depuis une String

```java
// Stream de caractères (IntStream)
IntStream caracteres = "Hello".chars();

// Stream de lignes
Stream<String> lignes = "ligne1\nligne2\nligne3".lines();
```

---

## Opérations intermédiaires

Les opérations intermédiaires transforment un Stream en un autre Stream. Elles sont **paresseuses** : elles ne font rien tant qu'une opération terminale n'est pas appelée.

### filter() : filtrer les éléments

Garde uniquement les éléments qui satisfont un prédicat :

```java
List<Integer> nombres = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

List<Integer> pairs = nombres.stream()
    .filter(n -> n % 2 == 0)
    .toList();

System.out.println(pairs); // [2, 4, 6, 8, 10]
```

### map() : transformer les éléments

Applique une fonction à chaque élément et retourne un nouveau Stream :

```java
List<String> noms = List.of("alice", "bob", "charlie");

List<String> majuscules = noms.stream()
    .map(String::toUpperCase)
    .toList();

System.out.println(majuscules); // [ALICE, BOB, CHARLIE]
```

On peut changer le type des éléments :

```java
List<String> mots = List.of("Java", "Stream", "API");

List<Integer> longueurs = mots.stream()
    .map(String::length)
    .toList();

System.out.println(longueurs); // [4, 6, 3]
```

### flatMap() : aplatir des structures imbriquées

`flatMap()` transforme chaque élément en un Stream, puis fusionne tous ces Streams en un seul :

```java
List<List<String>> listes = List.of(
    List.of("a", "b"),
    List.of("c", "d"),
    List.of("e")
);

List<String> aplatie = listes.stream()
    .flatMap(Collection::stream)
    .toList();

System.out.println(aplatie); // [a, b, c, d, e]
```

Cas d'usage courant : extraire des éléments depuis des objets contenant des listes :

```java
record Commande(String client, List<String> produits) {}

List<Commande> commandes = List.of(
    new Commande("Alice", List.of("Livre", "Stylo")),
    new Commande("Bob", List.of("Cahier"))
);

List<String> tousProduits = commandes.stream()
    .flatMap(c -> c.produits().stream())
    .toList();

System.out.println(tousProduits); // [Livre, Stylo, Cahier]
```

### sorted() : trier les éléments

Trie les éléments selon l'ordre naturel ou un `Comparator` :

```java
List<String> noms = List.of("Charlie", "Alice", "Bob");

// Tri naturel (alphabétique)
List<String> tries = noms.stream()
    .sorted()
    .toList();
// [Alice, Bob, Charlie]

// Tri par longueur
List<String> parLongueur = noms.stream()
    .sorted(Comparator.comparingInt(String::length))
    .toList();
// [Bob, Alice, Charlie]

// Tri inverse
List<String> inverse = noms.stream()
    .sorted(Comparator.reverseOrder())
    .toList();
// [Charlie, Bob, Alice]
```

### distinct() : supprimer les doublons

```java
List<Integer> nombres = List.of(1, 2, 2, 3, 3, 3, 4);

List<Integer> uniques = nombres.stream()
    .distinct()
    .toList();

System.out.println(uniques); // [1, 2, 3, 4]
```

> `distinct()` utilise `equals()` pour comparer les éléments. Si vous utilisez des objets personnalisés, pensez à implémenter `equals()` et `hashCode()`.

### limit() et skip() : découper le Stream

```java
List<Integer> nombres = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

// Les 3 premiers éléments
List<Integer> premiers = nombres.stream()
    .limit(3)
    .toList();
// [1, 2, 3]

// Sauter les 3 premiers
List<Integer> saufPremiers = nombres.stream()
    .skip(3)
    .toList();
// [4, 5, 6, 7, 8, 9, 10]

// Pagination : page 2, taille 3
List<Integer> page2 = nombres.stream()
    .skip(3)
    .limit(3)
    .toList();
// [4, 5, 6]
```

### peek() : observer sans modifier

`peek()` exécute une action sur chaque élément sans le modifier. Utile pour le debug :

```java
List<String> resultat = List.of("alice", "bob", "charlie").stream()
    .filter(nom -> nom.length() > 3)
    .peek(nom -> System.out.println("Après filter : " + nom))
    .map(String::toUpperCase)
    .peek(nom -> System.out.println("Après map : " + nom))
    .toList();
```

**Sortie :**
```
Après filter : alice
Après map : ALICE
Après filter : charlie
Après map : CHARLIE
```

> `peek()` est prévu pour le debug. Évitez d'y mettre de la logique métier.

---

## Opérations terminales

Les opérations terminales déclenchent le traitement du pipeline et produisent un résultat (une valeur, une collection) ou un effet de bord (`forEach`).

### collect() : collecter dans une structure

`collect()` est l'opération terminale la plus puissante. Elle utilise un `Collector` pour assembler les résultats :

```java
List<String> noms = List.of("Alice", "Bob", "Charlie");

// En List
List<String> liste = noms.stream()
    .filter(n -> n.length() > 3)
    .collect(Collectors.toList());

// En Set (supprime les doublons)
Set<String> ensemble = noms.stream()
    .collect(Collectors.toSet());

// En Map (clé → valeur)
Map<String, Integer> longueurs = noms.stream()
    .collect(Collectors.toMap(
        nom -> nom,           // clé
        String::length        // valeur
    ));
// {Alice=5, Bob=3, Charlie=7}
```

### toList() : raccourci depuis Java 16

Depuis Java 16, `toList()` remplace avantageusement `collect(Collectors.toList())` :

```java
// Avant Java 16
List<String> liste = noms.stream()
    .filter(n -> n.length() > 3)
    .collect(Collectors.toList());

// Depuis Java 16
List<String> liste = noms.stream()
    .filter(n -> n.length() > 3)
    .toList();
```

> `toList()` retourne une liste **immuable**. Si vous avez besoin d'une liste modifiable, utilisez `collect(Collectors.toList())`.

### Collectors utiles

Voici les Collectors les plus courants :

```java
List<String> noms = List.of("Alice", "Bob", "Charlie", "Alice");

// Joindre en une seule String
String joined = noms.stream()
    .collect(Collectors.joining(", "));
// "Alice, Bob, Charlie, Alice"

// Joindre avec préfixe et suffixe
String formatted = noms.stream()
    .collect(Collectors.joining(", ", "[", "]"));
// "[Alice, Bob, Charlie, Alice]"

// Compter
long count = noms.stream()
    .collect(Collectors.counting());
// 4

// Grouper (voir l'article dédié pour plus de détails)
Map<Integer, List<String>> parLongueur = noms.stream()
    .collect(Collectors.groupingBy(String::length));
// {3=[Bob], 5=[Alice, Alice], 7=[Charlie]}

// Partitionner (true/false)
Map<Boolean, List<String>> partition = noms.stream()
    .collect(Collectors.partitioningBy(n -> n.length() > 4));
// {false=[Bob], true=[Alice, Charlie, Alice]}
```

### forEach() : exécuter une action

```java
List.of("Alice", "Bob", "Charlie").stream()
    .filter(n -> n.length() > 3)
    .forEach(System.out::println);
// Alice
// Charlie
```

> `forEach()` est une opération terminale. Après son appel, le Stream est consommé.

### reduce() : agréger en une seule valeur

`reduce()` combine tous les éléments en un seul résultat :

```java
List<Integer> nombres = List.of(1, 2, 3, 4, 5);

// Somme
int somme = nombres.stream()
    .reduce(0, Integer::sum);
// 15

// Produit
int produit = nombres.stream()
    .reduce(1, (a, b) -> a * b);
// 120

// Sans valeur initiale (retourne Optional)
Optional<Integer> max = nombres.stream()
    .reduce(Integer::max);
// Optional[5]
```

**Concaténation de chaînes :**

```java
String phrase = List.of("Les", "Streams", "sont", "puissants").stream()
    .reduce("", (a, b) -> a + " " + b)
    .trim();
// "Les Streams sont puissants"
```

### count(), min(), max()

```java
List<Integer> nombres = List.of(3, 1, 4, 1, 5, 9, 2, 6);

long count = nombres.stream().count();                        // 8
Optional<Integer> min = nombres.stream().min(Integer::compare); // Optional[1]
Optional<Integer> max = nombres.stream().max(Integer::compare); // Optional[9]
```

### findFirst() et findAny()

```java
List<String> noms = List.of("Alice", "Bob", "Charlie");

Optional<String> premier = noms.stream()
    .filter(n -> n.startsWith("C"))
    .findFirst();
// Optional[Charlie]

Optional<String> nimporte = noms.stream()
    .filter(n -> n.length() > 3)
    .findAny();
// Optional[Alice] (non déterministe en parallèle)
```

### anyMatch(), allMatch(), noneMatch()

```java
List<Integer> nombres = List.of(2, 4, 6, 8);

boolean auMoinsUnImpair = nombres.stream().anyMatch(n -> n % 2 != 0);  // false
boolean tousPairs = nombres.stream().allMatch(n -> n % 2 == 0);         // true
boolean aucunNegatif = nombres.stream().noneMatch(n -> n < 0);          // true
```

Ces opérations sont **court-circuitantes** : elles s'arrêtent dès que le résultat est déterminé.

---

## Streams de primitifs

Pour éviter le coût du boxing/unboxing (`int` ↔ `Integer`), Java fournit des Streams spécialisés pour les types primitifs.

### IntStream, LongStream, DoubleStream

```java
// Depuis une plage de valeurs
IntStream.range(0, 5).forEach(System.out::print);     // 01234
IntStream.rangeClosed(1, 5).forEach(System.out::print); // 12345

// Depuis un tableau
int[] tableau = {1, 2, 3, 4, 5};
IntStream stream = Arrays.stream(tableau);

// Depuis un Stream d'objets avec mapToInt
List<String> mots = List.of("Java", "Stream", "API");
IntStream longueurs = mots.stream().mapToInt(String::length);
```

### Opérations spécifiques aux primitifs

Les Streams de primitifs offrent des méthodes d'agrégation directes, sans passer par des Collectors :

```java
int[] nombres = {3, 1, 4, 1, 5, 9, 2, 6};

int somme = IntStream.of(nombres).sum();                        // 31
OptionalInt min = IntStream.of(nombres).min();                   // OptionalInt[1]
OptionalInt max = IntStream.of(nombres).max();                   // OptionalInt[9]
OptionalDouble moyenne = IntStream.of(nombres).average();        // OptionalDouble[3.875]
IntSummaryStatistics stats = IntStream.of(nombres).summaryStatistics();

System.out.printf("count=%d, sum=%d, min=%d, max=%d, avg=%.2f%n",
    stats.getCount(), stats.getSum(), stats.getMin(), stats.getMax(), stats.getAverage());
// count=8, sum=31, min=1, max=9, avg=3.88
```

### Conversion entre Stream et IntStream

```java
// Stream<Integer> → IntStream
Stream<Integer> boxed = Stream.of(1, 2, 3);
IntStream primitif = boxed.mapToInt(Integer::intValue);

// IntStream → Stream<Integer>
IntStream primitif2 = IntStream.of(1, 2, 3);
Stream<Integer> boxed2 = primitif2.boxed();

// IntStream → List<Integer>
List<Integer> liste = IntStream.rangeClosed(1, 5)
    .boxed()
    .toList();
```

---

## Cas pratique : traitement de données

Mettons tout cela en pratique avec un exemple concret de traitement de données commerciales.

```java
record Produit(String nom, String categorie, double prix, int stock) {}

List<Produit> produits = List.of(
    new Produit("Laptop", "Électronique", 999.99, 50),
    new Produit("Souris", "Électronique", 29.99, 200),
    new Produit("Livre Java", "Livres", 45.00, 100),
    new Produit("Clavier", "Électronique", 79.99, 150),
    new Produit("Livre Python", "Livres", 39.99, 80),
    new Produit("Écran", "Électronique", 349.99, 30),
    new Produit("Livre SQL", "Livres", 35.00, 60)
);
```

### Filtrer et trier

Trouver les produits électroniques de moins de 100 euros, triés par prix :

```java
List<Produit> electroniquePasCher = produits.stream()
    .filter(p -> "Électronique".equals(p.categorie()))
    .filter(p -> p.prix() < 100)
    .sorted(Comparator.comparingDouble(Produit::prix))
    .toList();

electroniquePasCher.forEach(p ->
    System.out.printf("%s : %.2f EUR%n", p.nom(), p.prix())
);
// Souris : 29.99 EUR
// Clavier : 79.99 EUR
```

### Calculer des statistiques

```java
// Valeur totale du stock
double valeurStock = produits.stream()
    .mapToDouble(p -> p.prix() * p.stock())
    .sum();
System.out.printf("Valeur totale du stock : %.2f EUR%n", valeurStock);

// Prix moyen par catégorie
Map<String, Double> prixMoyenParCategorie = produits.stream()
    .collect(Collectors.groupingBy(
        Produit::categorie,
        Collectors.averagingDouble(Produit::prix)
    ));
System.out.println(prixMoyenParCategorie);
// {Électronique=364.99, Livres=39.99}
```

### Transformer en Map

Créer un dictionnaire nom → prix pour les produits en stock :

```java
Map<String, Double> catalogue = produits.stream()
    .filter(p -> p.stock() > 0)
    .collect(Collectors.toMap(
        Produit::nom,
        Produit::prix
    ));
```

### Trouver un produit

```java
Optional<Produit> moinsCher = produits.stream()
    .min(Comparator.comparingDouble(Produit::prix));

moinsCher.ifPresent(p ->
    System.out.printf("Le moins cher : %s à %.2f EUR%n", p.nom(), p.prix())
);
// Le moins cher : Souris à 29.99 EUR
```

---

## Streams parallèles

Les Streams parallèles répartissent le travail sur plusieurs threads pour exploiter les processeurs multi-coeurs.

### Créer un Stream parallèle

```java
// Depuis une collection
List<Integer> nombres = List.of(1, 2, 3, 4, 5);
Stream<Integer> parallel = nombres.parallelStream();

// Depuis un Stream existant
Stream<Integer> parallel2 = nombres.stream().parallel();
```

### Quand utiliser les Streams parallèles

Les Streams parallèles ne sont **pas** toujours plus rapides. Ils sont utiles quand :

| Critère             | Séquentiel       | Parallèle                         |
|---------------------|------------------|-----------------------------------|
| Quantité de données | Petite à moyenne | Grande (dizaines de milliers+)    |
| Coût par élément    | Faible           | Élevé (calcul, I/O)               |
| Source              | Toute            | Bien divisible (ArrayList, int[]) |
| Ordre important     | Oui              | Non                               |
| Effets de bord      | Acceptables      | Interdits                         |

```java
// Bon candidat : beaucoup de données, calcul coûteux
long count = IntStream.rangeClosed(1, 10_000_000)
    .parallel()
    .filter(n -> estPremier(n))
    .count();
```

### Pièges des Streams parallèles

```java
// ❌ Mauvais : effets de bord partagés
List<String> resultats = new ArrayList<>();
noms.parallelStream()
    .filter(n -> n.length() > 3)
    .forEach(resultats::add); // ConcurrentModificationException possible !

// ✅ Correct : utiliser collect
List<String> resultats = noms.parallelStream()
    .filter(n -> n.length() > 3)
    .collect(Collectors.toList());
```

---

## Pièges courants

### Un Stream ne peut être consommé qu'une fois

```java
Stream<String> stream = List.of("a", "b", "c").stream();

stream.forEach(System.out::println); // OK
stream.forEach(System.out::println); // IllegalStateException !
```

Si vous avez besoin de réutiliser un pipeline, créez un nouveau Stream à chaque fois ou stockez la source dans une variable.

### Ne pas modifier la source pendant le traitement

```java
List<String> noms = new ArrayList<>(List.of("Alice", "Bob", "Charlie"));

// ❌ Modification de la source pendant le stream
noms.stream()
    .filter(n -> n.length() > 3)
    .forEach(n -> noms.remove(n)); // ConcurrentModificationException

// ✅ Collecter d'abord, puis modifier
List<String> aSupprimer = noms.stream()
    .filter(n -> n.length() > 3)
    .toList();
noms.removeAll(aSupprimer);
```

### Attention au coût de sorted() et distinct()

`sorted()` doit conserver **tous les éléments** en mémoire avant de produire un résultat. Sur un Stream très volumineux ou infini, cela peut causer un `OutOfMemoryError` :

```java
// ❌ Dangereux : sorted sur un Stream infini
Stream.generate(Math::random)
    .sorted()  // Attend tous les éléments... qui ne finissent jamais
    .limit(10)
    .forEach(System.out::println);

// ✅ Limiter d'abord
Stream.generate(Math::random)
    .limit(10)
    .sorted()
    .forEach(System.out::println);
```

### toMap() et les clés en doublon

```java
List<String> noms = List.of("Alice", "Bob", "Amy");

// ❌ Clé en doublon → IllegalStateException
Map<Character, String> parInitiale = noms.stream()
    .collect(Collectors.toMap(
        n -> n.charAt(0),  // Alice et Amy ont la même initiale 'A'
        n -> n
    ));

// ✅ Gérer les conflits avec un merge function
Map<Character, String> parInitiale = noms.stream()
    .collect(Collectors.toMap(
        n -> n.charAt(0),
        n -> n,
        (existant, nouveau) -> existant + ", " + nouveau
    ));
// {A=Alice, Amy, B=Bob}
```

---

## FAQ

**Quelle est la différence entre `map()` et `flatMap()` ?**

`map()` transforme chaque élément en **un** autre élément (1 → 1). `flatMap()` transforme chaque élément en **un Stream** d'éléments, puis fusionne tous ces Streams (1 → N). Utilisez `flatMap()` quand vous avez des structures imbriquées à aplatir (listes de listes, Optional dans un Stream, etc.).

**`toList()` ou `collect(Collectors.toList())` ?**

`toList()` (Java 16+) est plus concis mais retourne une liste **immuable**. `collect(Collectors.toList())` retourne une `ArrayList` modifiable. Choisissez selon que vous avez besoin de modifier la liste après.

**Les Streams sont-ils toujours plus performants que les boucles ?**

Non. Pour des opérations simples sur de petites collections, les boucles sont équivalentes voire plus rapides (pas d'overhead d'objets intermédiaires). Les Streams brillent en lisibilité et en composabilité, pas en performance brute.

**Peut-on utiliser des Streams sur une Map ?**

Pas directement, mais on peut streamer les entrées :

```java
Map<String, Integer> scores = Map.of("Alice", 95, "Bob", 87, "Charlie", 92);

scores.entrySet().stream()
    .filter(e -> e.getValue() > 90)
    .forEach(e -> System.out.println(e.getKey() + " : " + e.getValue()));
```

---

## Conclusion

L'API Stream transforme la manière d'écrire du code Java en remplaçant les boucles impératives par des pipelines déclaratifs. Le code est plus concis, plus lisible et plus facile à composer.

**Points clés à retenir :**

- Un Stream est un pipeline de traitement, pas une structure de données
- Les opérations intermédiaires (`filter`, `map`, `sorted`) sont paresseuses
- Les opérations terminales (`collect`, `forEach`, `reduce`) déclenchent le pipeline
- Utilisez `toList()` (Java 16+) pour le cas simple, `collect()` pour les cas avancés
- Les Streams de primitifs (`IntStream`, `DoubleStream`) évitent le coût du boxing
- Les Streams parallèles ne sont utiles que pour de grands volumes avec des opérations coûteuses
- Un Stream ne s'utilise qu'une seule fois

L'API Stream est la base de nombreuses fonctionnalités du Java moderne : `Collectors.groupingBy()`, `Optional.stream()`, ou encore les Virtual Threads avec des patterns de concurrence. Maîtrisez-la pour tirer le meilleur du Java moderne.

---

## Pour aller plus loin

- [Stream Javadoc (Java 17)](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/Stream.html)
- [Collectors Javadoc (Java 17)](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/Collectors.html)
- [Package java.util.stream (Java 17)](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/package-summary.html)

## Voir aussi

- [Comment faire des group by en Java]({% post_url 2026-01-11-Comment-faire-des-group-by-en-Java %})
- [Optional en Java : éviter les NullPointerException]({% post_url 2026-01-26-Optional-en-Java-eviter-les-NullPointerException %})
- [Introduction aux collections Java]({% post_url 2020-11-12-Framework-collections-java-intro %})
- [Les listes (List) en Java]({% post_url 2025-09-19-Framework-collections-java-list %})
- [Les maps (Map) en Java]({% post_url 2025-10-04-Framework-collections-java-map %})
