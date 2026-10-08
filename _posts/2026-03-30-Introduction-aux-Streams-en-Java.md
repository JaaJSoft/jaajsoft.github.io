---
layout: article
title: "Introduction aux Streams en Java"
description: "Introduction à l'API Stream de Java : créer un Stream, opérations intermédiaires et terminales, Streams de primitifs, Collectors et Streams parallèles."
author: Pierre Chopinet
tags:
  - java
  - streams
  - collections
  - functional
---

L'API Stream, arrivée avec Java 8, permet d'enchaîner des traitements sur une collection (filtrer, transformer, trier, agréger) sans écrire la boucle soi-même. Dans ce tutoriel, nous allons voir comment créer un Stream, puis les opérations les plus utilisées et les quelques comportements qui surprennent la première fois.
<!--more-->

Dans cet article :
- Le principe d'un Stream
- Créer un Stream
- Les opérations intermédiaires
- Les opérations terminales
- Les Streams de primitifs
- Traiter une liste de produits
- Les Streams parallèles

Pré-requis : Java 8 pour l'API Stream, Java 16 pour `Stream.toList()` et les records utilisés dans les exemples. Les exemples ont été testés avec Java 21.

## Le principe d'un Stream

Un Stream décrit une suite d'opérations à appliquer à une source de données (une collection, un tableau, un fichier...). Contrairement à une collection, il ne stocke aucun élément :

```java
List<String> noms = List.of("Alice", "Bob", "Charlie", "David", "Eve");

List<String> resultat = noms.stream()
    .filter(nom -> nom.length() > 3)
    .map(String::toUpperCase)
    .sorted()
    .toList();

System.out.println(resultat); // [ALICE, CHARLIE, DAVID]
```

On garde les noms de plus de 3 caractères, on les met en majuscules, puis on les trie. Avec une boucle, il aurait fallu une liste intermédiaire, un `if` et un appel à `sort`.

Un Stream ne rend pas pour autant le code plus rapide. Sur une petite collection et un traitement simple, une boucle `for` fait aussi bien, voire un peu mieux. L'intérêt des Streams est la lisibilité quand plusieurs traitements s'enchaînent.

La source n'est pas modifiée : `noms` contient toujours ses cinq prénoms, le résultat est une nouvelle liste. Un Stream peut aussi être infini (on le verra avec `Stream.generate()`), et surtout, il est paresseux.

### L'évaluation paresseuse

Les opérations intermédiaires comme `filter` ou `map` ne s'exécutent pas tout de suite. Elles sont seulement ajoutées au pipeline, qui ne démarre qu'avec l'opération terminale :

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

Ce qui affiche :

```
Avant l'opération terminale
Filtrage de : Alice
Filtrage de : Bob
Filtrage de : Charlie
```

Les messages de filtrage n'apparaissent qu'au moment du `toList()`.

### Un Stream ne se consomme qu'une fois

Une fois l'opération terminale exécutée, le Stream est consommé. Le réutiliser lève une exception :

```java
Stream<String> stream = List.of("a", "b", "c").stream();

stream.forEach(System.out::println); // OK
stream.forEach(System.out::println); // IllegalStateException
```

Le message est explicite : `stream has already been operated upon or closed`. Pour refaire un traitement, on crée un nouveau Stream à partir de la source.

## Créer un Stream

### Depuis une collection ou un tableau

La méthode `stream()` est disponible sur toutes les collections. Pour un tableau, on passe par `Arrays.stream()`, et pour quelques valeurs, par `Stream.of()` :

```java
List<String> liste = List.of("a", "b", "c");
Stream<String> stream = liste.stream();

Set<Integer> ensemble = Set.of(1, 2, 3);
Stream<Integer> stream2 = ensemble.stream();

String[] tableau = {"a", "b", "c"};
Stream<String> stream3 = Arrays.stream(tableau);

Stream<String> stream4 = Stream.of("Alice", "Bob", "Charlie");
```

### Depuis une Map

Une `Map` n'a pas de méthode `stream()`, mais ses vues `entrySet()`, `keySet()` et `values()` en ont une :

```java
Map<String, Integer> scores = Map.of("Alice", 95, "Bob", 87, "Charlie", 92);

scores.entrySet().stream()
    .filter(e -> e.getValue() > 90)
    .forEach(e -> System.out.println(e.getKey() + " : " + e.getValue()));
```

```
Alice : 95
Charlie : 92
```

Attention, l'ordre d'itération d'une `Map` créée avec `Map.of()` n'est pas garanti, et il change même d'une exécution à l'autre : on peut tout aussi bien obtenir `Charlie` avant `Alice`.

### Des Streams infinis avec generate et iterate

`Stream.generate()` et `Stream.iterate()` produisent des Streams potentiellement infinis :

```java
// Génère une séquence infinie de nombres aléatoires
Stream<Double> aleatoires = Stream.generate(Math::random);

// Génère 0, 2, 4, 6, 8, ...
Stream<Integer> pairs = Stream.iterate(0, n -> n + 2);

// Avec une condition d'arrêt (Java 9+) : 0, 2, 4, ..., 18
Stream<Integer> pairsLimites = Stream.iterate(0, n -> n < 20, n -> n + 2);
```

Un Stream infini doit être limité avec `limit()`, ou avec une condition d'arrêt comme dans le dernier exemple, sinon l'opération terminale ne se termine jamais.

### Depuis un fichier ou une chaîne

`Files.lines()` lit un fichier ligne par ligne, au fur et à mesure du traitement. Le fichier reste ouvert tant que le Stream n'est pas fermé, d'où le `try` avec ressource, que la Javadoc demande explicitement :

```java
try (Stream<String> lignes = Files.lines(Path.of("fichier.txt"))) {
    lignes.filter(l -> !l.isBlank())
        .forEach(System.out::println);
}
```

Une chaîne de caractères peut aussi servir de source :

```java
// Stream de caractères (IntStream)
IntStream caracteres = "Hello".chars();

// Stream de lignes (String.lines() est disponible depuis Java 11)
Stream<String> lignes = "ligne1\nligne2\nligne3".lines();
```

## Les opérations intermédiaires

Les opérations intermédiaires transforment un Stream en un autre Stream. Comme on l'a vu, elles ne font rien tant qu'aucune opération terminale n'est appelée.

### filter() : filtrer les éléments

`filter()` garde les éléments qui respectent une condition :

```java
List<Integer> nombres = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

List<Integer> pairs = nombres.stream()
    .filter(n -> n % 2 == 0)
    .toList();

System.out.println(pairs); // [2, 4, 6, 8, 10]
```

### map() : transformer les éléments

`map()` applique une fonction à chaque élément :

```java
List<String> noms = List.of("alice", "bob", "charlie");

List<String> majuscules = noms.stream()
    .map(String::toUpperCase)
    .toList();

System.out.println(majuscules); // [ALICE, BOB, CHARLIE]
```

La fonction peut changer le type des éléments :

```java
List<String> mots = List.of("Java", "Stream", "API");

List<Integer> longueurs = mots.stream()
    .map(String::length)
    .toList();

System.out.println(longueurs); // [4, 6, 3]
```

### flatMap() : aplatir des structures imbriquées

`map()` transforme chaque élément en un autre élément. `flatMap()`, lui, transforme chaque élément en un Stream, puis fusionne tous ces Streams en un seul. On s'en sert pour aplatir une liste de listes :

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

Ou pour récupérer les éléments des listes contenues dans des objets :

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

`sorted()` trie selon l'ordre naturel, ou selon un `Comparator` :

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

Pour trier, `sorted()` doit d'abord récupérer tous les éléments. Sur un Stream infini, il attend donc indéfiniment, jusqu'à l'`OutOfMemoryError`. L'ordre des opérations compte :

```java
// À éviter : sorted sur un Stream infini
Stream.generate(Math::random)
    .sorted()  // Attend tous les éléments... qui ne finissent jamais
    .limit(10)
    .forEach(System.out::println);

// Correct : limiter d'abord
Stream.generate(Math::random)
    .limit(10)
    .sorted()
    .forEach(System.out::println);
```

### distinct() : supprimer les doublons

```java
List<Integer> nombres = List.of(1, 2, 2, 3, 3, 3, 4);

List<Integer> uniques = nombres.stream()
    .distinct()
    .toList();

System.out.println(uniques); // [1, 2, 3, 4]
```

`distinct()` compare les éléments avec `equals()`. Pour vos propres classes, il faut donc implémenter `equals()` et `hashCode()`, ce que les records font automatiquement.

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

`peek()` exécute une action sur chaque élément qui passe, sans le modifier. Il est pratique pour comprendre ce que fait un pipeline :

```java
List<String> resultat = List.of("alice", "bob", "charlie").stream()
    .filter(nom -> nom.length() > 3)
    .peek(nom -> System.out.println("Après filter : " + nom))
    .map(String::toUpperCase)
    .peek(nom -> System.out.println("Après map : " + nom))
    .toList();
```

On obtient :

```
Après filter : alice
Après map : ALICE
Après filter : charlie
Après map : CHARLIE
```

La sortie montre que les éléments traversent le pipeline un par un : `alice` passe le filtre et la transformation avant que `bob` ne soit examiné.

Par contre, `peek()` est fait pour le débogage, pas pour de la logique métier : le Stream peut sauter son exécution. Par exemple, `List.of("a", "b", "c").stream().peek(System.out::println).count()` n'affiche rien. La taille de la liste est connue d'avance, `count()` n'a donc pas besoin de parcourir les éléments, ce que la Javadoc de `count()` autorise explicitement.

## Les opérations terminales

Les opérations terminales déclenchent le pipeline et produisent un résultat (une valeur, une collection) ou un effet de bord (`forEach`).

### collect() : collecter dans une structure

`collect()` assemble les éléments à l'aide d'un `Collector`. La classe `Collectors` en fournit pour les cas courants :

```java
List<String> noms = List.of("Alice", "Bob", "Charlie");

// En List
List<String> liste = noms.stream()
    .filter(n -> n.length() > 3)
    .collect(Collectors.toList());

// En Set (supprime les doublons)
Set<String> ensemble = noms.stream()
    .collect(Collectors.toSet());

// En Map (clé -> valeur)
Map<String, Integer> longueurs = noms.stream()
    .collect(Collectors.toMap(
        nom -> nom,           // clé
        String::length        // valeur
    ));
System.out.println(longueurs);
```

```
{Bob=3, Alice=5, Charlie=7}
```

`toMap()` renvoie ici une `HashMap`, dont l'ordre d'itération n'est pas garanti : `Bob` sort avant `Alice`.

Attention aussi aux clés en double. Si deux éléments donnent la même clé, `toMap()` lève une exception :

```java
List<String> noms = List.of("Alice", "Bob", "Amy");

// À éviter : clé en doublon -> IllegalStateException
Map<Character, String> parInitiale = noms.stream()
    .collect(Collectors.toMap(
        n -> n.charAt(0),  // Alice et Amy ont la même initiale 'A'
        n -> n
    ));
```

```
java.lang.IllegalStateException: Duplicate key A (attempted merging values Alice and Amy)
```

Il faut alors dire à `toMap()` comment fusionner les deux valeurs, avec un troisième paramètre :

```java
// Correct : gérer les conflits avec une fonction de fusion
Map<Character, String> parInitiale = noms.stream()
    .collect(Collectors.toMap(
        n -> n.charAt(0),
        n -> n,
        (existant, nouveau) -> existant + ", " + nouveau
    ));
// {A=Alice, Amy, B=Bob}
```

### toList() : obtenir une liste directement

Depuis Java 16, `toList()` remplace `collect(Collectors.toList())` dans la plupart des cas :

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

Les deux ne sont pas tout à fait équivalents. La liste renvoyée par `toList()` est non modifiable : un `add()` lève une `UnsupportedOperationException`. Celle de `Collectors.toList()` est aujourd'hui une `ArrayList`, mais sa Javadoc ne garantit rien sur son type ni sur le fait qu'elle soit modifiable. Si vous avez besoin d'une liste modifiable, demandez-la explicitement avec `Collectors.toCollection(ArrayList::new)`.

### D'autres Collectors : joindre, compter, grouper

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

// Grouper
Map<Integer, List<String>> parLongueur = noms.stream()
    .collect(Collectors.groupingBy(String::length));
// {3=[Bob], 5=[Alice, Alice], 7=[Charlie]}

// Partitionner (true/false)
Map<Boolean, List<String>> partition = noms.stream()
    .collect(Collectors.partitioningBy(n -> n.length() > 4));
// {false=[Bob], true=[Alice, Charlie, Alice]}
```

`groupingBy()` mériterait un article à lui seul. Ça tombe bien, il y en a un : [Java : Comment faire des group by]({% post_url 2026-01-11-Comment-faire-des-group-by-en-Java %}).

### forEach() : exécuter une action

```java
List.of("Alice", "Bob", "Charlie").stream()
    .filter(n -> n.length() > 3)
    .forEach(System.out::println);
// Alice
// Charlie
```

Comme toute opération terminale, `forEach()` consomme le Stream. Attention à ne pas l'utiliser pour modifier la source du Stream :

```java
List<String> noms = new ArrayList<>(List.of("Alice", "Bob", "Charlie"));

// À éviter : modification de la source pendant le stream
noms.stream()
    .filter(n -> n.length() > 3)
    .forEach(n -> noms.remove(n));
```

On s'attendrait à une `ConcurrentModificationException`, mais ce code lève une `NullPointerException`, une fois `Alice` et `Charlie` supprimées de la liste. Le Stream continue de lire le tableau interne de l'`ArrayList` jusqu'à sa taille de départ : après les suppressions, il tombe sur une case vide, et le filtre reçoit `null`. Sans ce `null`, on aurait eu une `ConcurrentModificationException` à la fin du parcours. La documentation du package `java.util.stream` prévient d'ailleurs que modifier la source pendant l'exécution d'un pipeline peut provoquer des exceptions ou des résultats faux.

Pour supprimer des éléments d'une collection selon une condition, il y a plus simple :

```java
// Correct : removeIf, ou collecter d'abord puis supprimer
noms.removeIf(n -> n.length() > 3);
// [Bob]
```

### reduce() : agréger en une seule valeur

`reduce()` combine tous les éléments deux à deux pour obtenir un seul résultat :

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

Sans valeur initiale, il n'y a pas de résultat quand le Stream est vide, d'où l'`Optional`. Pour concaténer des chaînes, `Collectors.joining()` est plus adapté que `reduce()`.

### count(), min() et max() : compter et trouver les extrêmes

```java
List<Integer> nombres = List.of(3, 1, 4, 1, 5, 9, 2, 6);

long count = nombres.stream().count();                          // 8
Optional<Integer> min = nombres.stream().min(Integer::compare); // Optional[1]
Optional<Integer> max = nombres.stream().max(Integer::compare); // Optional[9]
```

### findFirst() et findAny() : récupérer un élément

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

### anyMatch(), allMatch() et noneMatch() : tester une condition

```java
List<Integer> nombres = List.of(2, 4, 6, 8);

boolean auMoinsUnImpair = nombres.stream().anyMatch(n -> n % 2 != 0);  // false
boolean tousPairs = nombres.stream().allMatch(n -> n % 2 == 0);         // true
boolean aucunNegatif = nombres.stream().noneMatch(n -> n < 0);          // true
```

Ces opérations s'arrêtent dès que le résultat est connu : `anyMatch()` n'examine pas la suite du Stream une fois qu'il a trouvé un élément qui convient.

## Les Streams de primitifs

Un `Stream<Integer>` manipule des objets `Integer`, ce qui oblige à convertir en permanence entre `int` et `Integer` (*boxing*). Pour l'éviter, Java fournit des Streams spécialisés : `IntStream`, `LongStream` et `DoubleStream`.

```java
// Depuis une plage de valeurs
IntStream.range(0, 5).forEach(System.out::print);       // 01234
IntStream.rangeClosed(1, 5).forEach(System.out::print); // 12345

// Depuis un tableau
int[] tableau = {1, 2, 3, 4, 5};
IntStream stream = Arrays.stream(tableau);

// Depuis un Stream d'objets avec mapToInt
List<String> mots = List.of("Java", "Stream", "API");
IntStream longueurs = mots.stream().mapToInt(String::length);
```

Ils ont des méthodes d'agrégation directes, sans passer par un `Collector` :

```java
int[] nombres = {3, 1, 4, 1, 5, 9, 2, 6};

int somme = IntStream.of(nombres).sum();                    // 31
OptionalInt min = IntStream.of(nombres).min();              // OptionalInt[1]
OptionalInt max = IntStream.of(nombres).max();              // OptionalInt[9]
OptionalDouble moyenne = IntStream.of(nombres).average();   // OptionalDouble[3.875]
IntSummaryStatistics stats = IntStream.of(nombres).summaryStatistics();

System.out.printf("count=%d, sum=%d, min=%d, max=%d, avg=%.2f%n",
    stats.getCount(), stats.getSum(), stats.getMin(), stats.getMax(), stats.getAverage());
// count=8, sum=31, min=1, max=9, avg=3.88
```

Pour passer d'un type de Stream à l'autre, on utilise `mapToInt()` dans un sens et `boxed()` dans l'autre :

```java
// Stream<Integer> -> IntStream
Stream<Integer> boxed = Stream.of(1, 2, 3);
IntStream primitif = boxed.mapToInt(Integer::intValue);

// IntStream -> Stream<Integer>
IntStream primitif2 = IntStream.of(1, 2, 3);
Stream<Integer> boxed2 = primitif2.boxed();

// IntStream -> List<Integer>
List<Integer> liste = IntStream.rangeClosed(1, 5)
    .boxed()
    .toList();
// [1, 2, 3, 4, 5]
```

## Traiter une liste de produits

Voici maintenant un exemple plus proche d'un vrai traitement, sur une liste de produits :

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

Les produits électroniques à moins de 100 euros, du moins cher au plus cher :

```java
List<Produit> electroniquePasCher = produits.stream()
    .filter(p -> "Électronique".equals(p.categorie()))
    .filter(p -> p.prix() < 100)
    .sorted(Comparator.comparingDouble(Produit::prix))
    .toList();

electroniquePasCher.forEach(p ->
    System.out.printf("%s : %.2f EUR%n", p.nom(), p.prix())
);
```

```
Souris : 29.99 EUR
Clavier : 79.99 EUR
```

`printf` utilise la locale par défaut de la JVM : sur un poste configuré en français, on obtient `29,99 EUR`.

La valeur totale du stock, puis le prix moyen par catégorie :

```java
double valeurStock = produits.stream()
    .mapToDouble(p -> p.prix() * p.stock())
    .sum();
System.out.printf("Valeur totale du stock : %.2f EUR%n", valeurStock);

Map<String, Double> prixMoyenParCategorie = produits.stream()
    .collect(Collectors.groupingBy(
        Produit::categorie,
        Collectors.averagingDouble(Produit::prix)
    ));
System.out.println(prixMoyenParCategorie);
```

```
Valeur totale du stock : 88294.90 EUR
{Livres=39.99666666666667, Électronique=364.99}
```

Une `Map` nom -> prix des produits en stock, et le produit le moins cher :

```java
Map<String, Double> catalogue = produits.stream()
    .filter(p -> p.stock() > 0)
    .collect(Collectors.toMap(
        Produit::nom,
        Produit::prix
    ));

Optional<Produit> moinsCher = produits.stream()
    .min(Comparator.comparingDouble(Produit::prix));

moinsCher.ifPresent(p ->
    System.out.printf("Le moins cher : %s à %.2f EUR%n", p.nom(), p.prix())
);
// Le moins cher : Souris à 29.99 EUR
```

## Les Streams parallèles

Un Stream parallèle répartit le traitement sur plusieurs threads, pour profiter des différents cœurs du processeur :

```java
// Depuis une collection
List<Integer> nombres = List.of(1, 2, 3, 4, 5);
Stream<Integer> parallel = nombres.parallelStream();

// Depuis un Stream existant
Stream<Integer> parallel2 = nombres.stream().parallel();
```

Ce n'est pas toujours plus rapide : découper le travail et rassembler les résultats a un coût. Un Stream parallèle devient intéressant quand il y a beaucoup d'éléments (des dizaines de milliers au moins) et que le traitement de chaque élément coûte cher. Il faut aussi que la source se découpe bien (une `ArrayList`, un tableau, un `IntStream.range()`), et que le traitement n'ait pas d'effet de bord. Compter les nombres premiers jusqu'à 10 millions est un bon candidat :

```java
static boolean estPremier(int n) {
    if (n < 2) return false;
    for (int i = 2; (long) i * i <= n; i++) {
        if (n % i == 0) return false;
    }
    return true;
}

long count = IntStream.rangeClosed(1, 10_000_000)
    .parallel()
    .filter(n -> estPremier(n))
    .count();
// 664579
```

Sur une VM à 4 vCPU avec Java 21, ce calcul prend environ 3,3 s en séquentiel et 1 s en parallèle.

Les effets de bord, justement, sont le principal piège. En parallèle, `forEach()` est exécuté par plusieurs threads en même temps, dans un ordre quelconque. Remplir une `ArrayList`, qui n'est pas thread-safe, depuis un `forEach()` parallèle donne des résultats faux :

```java
// À éviter : effets de bord partagés
List<Integer> resultats = new ArrayList<>();
IntStream.range(0, 100_000)
    .parallel()
    .boxed()
    .forEach(resultats::add);
System.out.println(resultats.size());
```

Selon les exécutions, on obtient une taille fausse (53 204 éléments au lieu de 100 000 lors d'un essai) ou une `ArrayIndexOutOfBoundsException`. La solution est de laisser le Stream construire le résultat :

```java
// Correct : laisser le Stream collecter
List<Integer> resultats = IntStream.range(0, 100_000)
    .parallel()
    .boxed()
    .toList();
// 100 000 éléments, dans l'ordre
```

Et voilà, vous avez maintenant de quoi remplacer une bonne partie de vos boucles par des Streams.

## Voir aussi

- [Java : Comment faire des group by]({% post_url 2026-01-11-Comment-faire-des-group-by-en-Java %})
- [Optional en Java : éviter les NullPointerException]({% post_url 2026-01-26-Optional-en-Java-eviter-les-NullPointerException %})
- [Introduction aux collections Java]({% post_url 2020-11-12-Framework-collections-java-intro %})
- [Les maps (Map) en Java]({% post_url 2025-10-04-Framework-collections-java-map %})
- [Javadoc de l'interface Stream (Java 17)](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/Stream.html)
- [Javadoc de la classe Collectors (Java 17)](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/Collectors.html)
- [Le package java.util.stream (Java 17)](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/package-summary.html)
