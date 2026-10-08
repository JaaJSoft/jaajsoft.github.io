---
layout: article
title: Les ensembles (Set) en Java
description: "Les ensembles en Java : interface Set, HashSet, LinkedHashSet, TreeSet et EnumSet, union et intersection, equals et hashCode, ensembles non modifiables."
author: Pierre Chopinet
tags:
  - java
  - collections
  - set
---

Troisième partie de notre série sur les collections Java, consacrée aux ensembles. Un `Set` garantit qu'un même élément n'y figure jamais deux fois : nous allons voir sur quoi repose cette garantie, comment choisir entre `HashSet`, `LinkedHashSet`, `TreeSet` et `EnumSet`, et comment calculer une union, une intersection ou une différence.
<!--more-->

1. [Introduction aux collections Java]({% post_url 2020-11-12-Framework-collections-java-intro %})
2. [Les listes (List) en Java]({% post_url 2025-09-19-Framework-collections-java-list %})
3. Les ensembles (Set) en Java (vous êtes ici)
4. [Les files (Queue) et Deques en Java]({% post_url 2025-09-26-Framework-collections-java-queue %})
5. [Les maps (Map) en Java]({% post_url 2025-10-04-Framework-collections-java-map %})
6. Utilisations avancées : [Introduction aux Streams en Java]({% post_url 2026-03-30-Introduction-aux-Streams-en-Java %}) et [Java : Comment faire des group by]({% post_url 2026-01-11-Comment-faire-des-group-by-en-Java %})

Les exemples ont été testés avec Java 21.

## Ce que garantit un Set

`Set` est une sous-interface de `Collection` qui représente un ensemble au sens mathématique : il ne contient jamais deux éléments égaux. Contrairement à une liste, il n'a pas d'index, et l'ordre dans lequel on retrouve les éléments en le parcourant dépend de l'implémentation.

```java
public interface Set<E> extends Collection<E> { /* ... */ }
```

`Set` n'ajoute aucune méthode à `Collection`, à part les fabriques statiques `of` et `copyOf` que nous verrons plus bas. Ce qui change, c'est le contrat des méthodes existantes. `add` n'ajoute l'élément que s'il n'est pas déjà présent, et le booléen retourné indique si l'ensemble a changé :

```java
Set<String> tags = new HashSet<>();

boolean added = tags.add("java");     // true si l'élément n'était pas présent
added = tags.add("java");             // false (doublon ignoré)

boolean present = tags.contains("java"); // true
boolean removed = tags.remove("java");   // true
```

De même, deux ensembles sont égaux au sens de `equals` s'ils contiennent les mêmes éléments, quel que soit l'ordre de parcours et quelle que soit leur implémentation : un `HashSet` et un `TreeSet` qui contiennent les mêmes chaînes sont égaux.

Reste à savoir quand deux éléments sont "les mêmes". Pour `HashSet` et `LinkedHashSet`, ce sont les méthodes `hashCode()` et `equals()` des éléments qui en décident. Pour `TreeSet`, c'est l'ordre de tri : deux éléments pour lesquels `compareTo` (ou le `compare` du comparateur) retourne 0 sont considérés comme un seul et même élément.

## Les implémentations

| Implémentation          | Structure                       | Ordre de parcours                   | `null`                      | `add`, `contains`, `remove` |
|-------------------------|---------------------------------|-------------------------------------|-----------------------------|-----------------------------|
| `HashSet`               | Table de hachage                | Aucun ordre garanti                 | Accepté                     | O(1)                        |
| `LinkedHashSet`         | Table de hachage + liste chaînée | Ordre d'insertion                   | Accepté                     | O(1)                        |
| `TreeSet`               | Arbre rouge-noir                | Trié                                | Refusé avec l'ordre naturel | O(log n)                    |
| `EnumSet`               | Vecteur de bits                 | Ordre de déclaration des constantes | Refusé                      | O(1)                        |
| `CopyOnWriteArraySet`   | Tableau recopié à chaque écriture | Ordre d'insertion                   | Accepté                     | O(n)                        |
| `ConcurrentSkipListSet` | Skip list                       | Trié                                | Refusé                      | O(log n) en moyenne         |

`HashSet` est le choix par défaut. Il s'appuie en réalité sur une `HashMap`, dont les éléments de l'ensemble sont les clés. `add`, `contains` et `remove` se font en temps constant, à condition que les `hashCode` des éléments soient bien répartis. En contrepartie, l'ordre de parcours n'est pas garanti, et rien ne dit qu'il restera le même au fil des ajouts.

`LinkedHashSet` ajoute à la table de hachage une liste doublement chaînée qui relie les éléments dans leur ordre d'insertion. Les performances restent proches de celles d'un `HashSet` (un peu en dessous, puisqu'il faut maintenir la liste), avec un ordre de parcours prévisible. Ajouter à nouveau un élément déjà présent ne change pas sa place.

`TreeSet` garde ses éléments triés selon leur ordre naturel ou selon le `Comparator` passé au constructeur. Il s'appuie sur une `TreeMap`, un arbre rouge-noir dont il utilise les clés. Ses opérations se font en O(log n). Il implémente aussi `NavigableSet`, qui permet de naviguer dans l'ordre de tri : `first()` et `last()` pour le premier et le dernier élément, `lower()`, `floor()`, `ceiling()` et `higher()` pour l'élément juste avant ou juste après une valeur, `headSet()`, `tailSet()` et `subSet()` pour une plage.

```java
TreeSet<Integer> notes = new TreeSet<>(List.of(12, 5, 18, 9, 15));
System.out.println(notes);              // [5, 9, 12, 15, 18]
System.out.println(notes.ceiling(10));  // 12 : le plus petit élément >= 10
System.out.println(notes.floor(10));    // 9 : le plus grand élément <= 10
System.out.println(notes.headSet(12));  // [5, 9] : les éléments < 12
System.out.println(notes.tailSet(12));  // [12, 15, 18] : les éléments >= 12
```

`EnumSet` est réservé aux constantes d'un même `enum`. Chaque constante y est représentée par un bit, ce qui le rend très compact et très rapide. C'est la bonne structure pour des options, des droits ou des états :

```java
enum Permission { READ, WRITE, DELETE, ADMIN }

EnumSet<Permission> droits = EnumSet.of(Permission.WRITE, Permission.READ);
System.out.println(droits);                       // [READ, WRITE]
System.out.println(EnumSet.complementOf(droits)); // [DELETE, ADMIN]
```

L'article sur [les enums en Java]({% post_url 2026-04-13-Les-Enums-en-Java-bien-plus-que-des-constantes %}) présente les autres façons de créer un `EnumSet` (`allOf`, `noneOf`, `range`).

Les deux dernières implémentations du tableau sont faites pour les programmes multithreads : nous les verrons à la fin de l'article.

## Union, intersection et différence

Les méthodes `addAll`, `retainAll` et `removeAll` correspondent aux opérations ensemblistes. Comme elles modifient l'ensemble sur lequel on les appelle, on travaille sur une copie pour garder les ensembles de départ intacts :

```java
Set<Integer> a = new HashSet<>(Set.of(1, 2, 3));
Set<Integer> b = new HashSet<>(Set.of(3, 4));

Set<Integer> union = new HashSet<>(a);
union.addAll(b);
System.out.println(union); // [1, 2, 3, 4]

Set<Integer> inter = new HashSet<>(a);
inter.retainAll(b);
System.out.println(inter); // [3]

Set<Integer> diff = new HashSet<>(a);
diff.removeAll(b);
System.out.println(diff);  // [1, 2]
```

Attention, l'ordre d'affichage d'un `HashSet` n'est pas garanti : ici les petits entiers sortent dans l'ordre croissant, mais c'est un effet de l'implémentation sur lequel il ne faut pas compter.

Pour savoir si un ensemble est inclus dans un autre, on utilise `containsAll` : `a.containsAll(inter)` retourne `true`, `a.containsAll(b)` retourne `false`.

## Trier avec un TreeSet

Avec un comparateur, un `TreeSet` peut maintenir un classement à jour. Ici, les joueurs sont triés par score décroissant, puis par nom :

```java
record User(String username, int score) {}

// Tri par score décroissant, puis username
Comparator<User> byScoreDescThenName =
        Comparator.comparingInt(User::score).reversed()
                  .thenComparing(User::username);

Set<User> leaderboard = new TreeSet<>(byScoreDescThenName);
leaderboard.add(new User("alice", 42));
leaderboard.add(new User("bob", 42));
leaderboard.add(new User("carl", 10));

// itération triée selon le comparateur
leaderboard.forEach(System.out::println);
```

Ce qui donne :

```
User[username=alice, score=42]
User[username=bob, score=42]
User[username=carl, score=10]
```

Le `thenComparing` ne sert pas seulement à classer les ex aequo par ordre alphabétique : sans lui, `bob` disparaîtrait du classement. Un `TreeSet` considère en effet que deux éléments sont égaux quand le comparateur retourne 0, et pour un comparateur basé uniquement sur le score, `alice` et `bob` sont égaux :

```java
Set<User> byScore = new TreeSet<>(Comparator.comparingInt(User::score).reversed());
byScore.add(new User("alice", 42));
byScore.add(new User("bob", 42));   // retourne false
byScore.add(new User("carl", 10));
System.out.println(byScore);
// [User[username=alice, score=42], User[username=carl, score=10]]
```

Dans cet ensemble, `byScore.contains(new User("zoe", 42))` retourne même `true`. Pour éviter ces surprises, le comparateur d'un `TreeSet` doit être cohérent avec `equals` : il ne doit retourner 0 que pour des éléments égaux.

## Les éléments doivent avoir un equals et un hashCode cohérents

Pour `HashSet` et `LinkedHashSet`, l'unicité repose sur `hashCode()` et `equals()`, ce qui impose deux règles.

D'abord, si une classe redéfinit `equals`, elle doit redéfinir `hashCode` de façon compatible : deux objets égaux doivent avoir le même `hashCode`. Sinon, deux objets égaux peuvent être rangés dans deux cases différentes de la table de hachage, et l'ensemble les garde tous les deux.

Ensuite, un élément ne doit pas être modifié d'une façon qui change son `hashCode` tant qu'il se trouve dans l'ensemble. Un `HashSet` range chaque élément dans une case calculée à partir de son `hashCode` au moment de l'ajout : si le `hashCode` change ensuite, l'élément est cherché dans la mauvaise case.

```java
class Person {
    String ssn; // utilisé dans equals/hashCode
    String name;

    @Override
    public boolean equals(Object o) {
        return o instanceof Person p && Objects.equals(ssn, p.ssn);
    }

    @Override
    public int hashCode() {
        return Objects.hashCode(ssn);
    }
}

Set<Person> s = new HashSet<>();
Person p = new Person();
p.ssn = "123";
s.add(p);

p.ssn = "999";                      // le hashCode change
System.out.println(s.contains(p));  // false
System.out.println(s.size());       // 1 : p est toujours dans le set...
System.out.println(s.remove(p));    // false : ... mais on ne peut plus le retirer
```

La solution est de ne baser `equals` et `hashCode` que sur des champs qui ne changent pas, ou mieux, d'utiliser des objets immuables. Les [records]({% post_url 2026-01-10-Records-en-Java-simplifier-vos-DTOs %}) s'y prêtent bien : leurs champs sont `final`, et `equals` et `hashCode` sont générés à partir de tous leurs composants.

## Dédoublonner une liste

Copier une liste dans un `LinkedHashSet` supprime les doublons tout en gardant l'ordre de première apparition des éléments :

```java
List<String> emails = List.of("a@x", "b@x", "a@x");
Set<String> unique = new LinkedHashSet<>(emails); // [a@x, b@x]
```

Et pour revenir à une liste sans doublons, dans le même ordre :

```java
List<String> dedup = new ArrayList<>(new LinkedHashSet<>(emails)); // [a@x, b@x]
```

Avec un `HashSet`, les doublons disparaissent aussi, mais l'ordre d'origine est perdu.

## Ensembles non modifiables

`Set.of(...)` (Java 9+) crée un ensemble non modifiable qui refuse les `null` et, plus surprenant, les doublons : `Set.of("ADMIN", "ADMIN")` lève une `IllegalArgumentException: duplicate element: ADMIN`. `Set.copyOf(collection)` (Java 10+), lui, accepte une collection qui contient des doublons et n'en garde qu'un exemplaire.

```java
Set<String> roles = Set.of("ADMIN", "USER");
Set<String> copie = Set.copyOf(List.of("ADMIN", "USER", "ADMIN")); // 2 éléments
```

L'ordre de parcours de ces ensembles n'est pas spécifié, et en pratique il change d'une exécution à l'autre : lancé plusieurs fois de suite, `System.out.println(Set.of("ADMIN", "USER", "GUEST", "ROOT"))` affiche tantôt `[ADMIN, GUEST, ROOT, USER]`, tantôt `[USER, ROOT, GUEST, ADMIN]`.

Pour exposer un ensemble interne en lecture seule, `Collections.unmodifiableSet` retourne une vue non modifiable, comme `unmodifiableList` pour les listes :

```java
class Service {
    private final Set<String> scopes = new HashSet<>();
    public Set<String> getScopes() {
        return Collections.unmodifiableSet(scopes);
    }
}
```

## Ensembles et threads

Comme les listes, `HashSet`, `LinkedHashSet` et `TreeSet` ne sont pas synchronisés. Si plusieurs threads partagent un ensemble, on peut utiliser `Collections.synchronizedSet(new HashSet<>())`, qui synchronise chaque méthode. Les parcours doivent alors se faire dans un bloc `synchronized` sur l'ensemble, comme pour `synchronizedList`.

`ConcurrentHashMap.newKeySet()` retourne un ensemble adossé à une `ConcurrentHashMap` : c'est l'équivalent concurrent d'un `HashSet`, sans verrou global, mais il refuse les `null`. De la même façon, `ConcurrentSkipListSet` est l'équivalent concurrent d'un `TreeSet`. Attention, sa méthode `size()` n'est pas en temps constant : elle doit parcourir tous les éléments.

Enfin, `CopyOnWriteArraySet` recopie son tableau interne à chaque modification. Comme ses éléments sont dans un simple tableau, `add` et `contains` le parcourent en entier : il est fait pour de petits ensembles, très souvent lus et rarement modifiés.

Dans la partie suivante, nous verrons les [files et les deques]({% post_url 2025-09-26-Framework-collections-java-queue %}), pour traiter des éléments dans leur ordre d'arrivée ou par priorité.

## Voir aussi

- [Les maps (Map) en Java]({% post_url 2025-10-04-Framework-collections-java-map %})
- [Les enums en Java]({% post_url 2026-04-13-Les-Enums-en-Java-bien-plus-que-des-constantes %})
- [Records en Java : simplifier vos DTOs]({% post_url 2026-01-10-Records-en-Java-simplifier-vos-DTOs %})
- [Javadoc de Set (Java 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Set.html)
- [Javadoc de HashSet (Java 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/HashSet.html)
- [Javadoc de LinkedHashSet (Java 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/LinkedHashSet.html)
- [Javadoc de TreeSet (Java 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/TreeSet.html)
- [Javadoc d'EnumSet (Java 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/EnumSet.html)
