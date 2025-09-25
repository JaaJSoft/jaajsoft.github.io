---
layout: article
title: Les ensembles (Set) en Java
author: Pierre Chopinet
tags:
  - java
  - collections
  - set
---

Dans cet article (partie 3 de la série sur les collections), nous allons nous concentrer sur la famille Set du Framework Collections : ses caractéristiques, ses principales implémentations (HashSet, LinkedHashSet, TreeSet, EnumSet…), leurs différences, pièges courants et bonnes pratiques d’utilisation.
<!--more-->

1. [Introduction aux collections Java]({% post_url 2020-11-12-Framework-collections-java-intro %})
2. [Les listes en Java]({% post_url 2025-09-19-Framework-collections-java-list %})
3. Les ensembles (Set) en Java (vous êtes ici)
4. Collections Queue (article à venir)
5. Collections Map (article à venir)
6. Utilisations avancées des collections (article à venir)

## Qu’est‑ce qu’un Set ?

`Set` est une sous‑interface de `Collection` qui représente un ensemble d’éléments sans doublon.

- Unicité des éléments (pas de doublons).
- Pas d’accès par index (contrairement à `List`).
- Notion d’ordre dépend de l’implémentation (pas ordre, selon l'ordre d’insertion, ou trié).

Signature :

```java
public interface Set<E> extends Collection<E> { /* … */ }
```

L’unicité est définie par :
- Pour les ensembles à base de hachage (`HashSet`, `LinkedHashSet`) : la combinaison de `hashCode()` et `equals()` des éléments.
- Pour les ensembles triés (`TreeSet`) : l’ordre imposé par `Comparator`/`Comparable` (deux éléments considérés « égaux » si `compare(a, b) == 0`).

## Méthodes de l’interface Set

En plus des méthodes héritées de `Collection`, voici les méthodes les plus utilisées avec leurs particularités pour `Set` :

| Méthode                                                | Description                                                                                             |
|--------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| boolean add(E e)                                       | Ajoute l’élément s’il n’est pas déjà présent. Retourne true si le set a changé (doublons ignorés).      |
| boolean addAll(Collection<? extends E> c)              | Ajoute les éléments non présents de c. Les doublons sont ignorés. Retourne true si le set a changé.     |
| boolean contains(Object o)                             | true si l’élément est présent (défini par equals/hashCode, ou par l’ordre du comparateur pour TreeSet). |
| boolean containsAll(Collection<?> c)                   | true si tous les éléments de c sont présents.                                                           |
| boolean remove(Object o)                               | Supprime l’élément s’il est présent. Retourne true si un élément a été retiré.                          |
| boolean removeAll(Collection<?> c)                     | Supprime tous les éléments présents dans c.                                                             |
| boolean retainAll(Collection<?> c)                     | Ne garde que les éléments aussi présents dans c (intersection).                                         |
| void clear()                                           | Vide le set.                                                                                            |
| int size()                                             | Nombre d’éléments.                                                                                      |
| boolean isEmpty()                                      | true si vide.                                                                                           |
| Iterator<E> iterator()                                 | Itérateur sur le set (ordre selon l’implémentation : insertion, aucun, trié).                           |
| Object[] toArray()                                     | Copie les éléments dans un tableau Object[].                                                            |
| <T> T[] toArray(T[] a)                                 | Copie les éléments dans un tableau typé fourni.                                                         |
| boolean equals(Object o)                               | Égalité de contenu (indépendante de l’ordre pour les ensembles).                                        |
| int hashCode()                                         | Hash basé sur les éléments (somme des hashCodes des éléments).                                          |
| static <E> Set<E> of(E... elements)                    | Crée un set immuable (Java 9+). Rejette les doublons (IllegalArgumentException).                        |
| static <E> Set<E> copyOf(Collection<? extends E> coll) | Crée une copie immuable d’une collection (Java 10+). Rejette les doublons.                              |

Remarques :
- `add`/`addAll` n’ajoutent jamais de doublons ; le booléen indique si le set a effectivement changé.
- Le contrat de `Set.equals` ne dépend pas de l’ordre d’itération : deux sets égaux contiennent les mêmes éléments.
- Les usines immuables (`Set.of`, `Set.copyOf`) rejettent toute tentative de doublon à la création.

## Principales implémentations

### HashSet

- Structure : table de hachage.
- Pas d’ordre garanti d’itération.
- Opérations de base (`add`, `contains`, `remove`) en `O(1)`.
- Autorise un seul `null`.

Cas d’usage : ensemble générique sans contrainte d’ordre, rapide par défaut.

### LinkedHashSet

- Même base que `HashSet` + chaîne de maillons pour l’ordre d’insertion (ou ordre d’accès si activé côté `LinkedHashMap`).
- Itération prévisible (ordre d’insertion).
- Surcoût mémoire minime par rapport à `HashSet`.

Cas d’usage : vous voulez la rapidité d’un `HashSet` avec un ordre d’itération stable (journalisation, export, UI).

### TreeSet (NavigableSet)

- Structure : arbre rouge‑noir (trié).
- Itération triée selon l’ordre naturel ou un `Comparator` fourni au constructeur.
- Opérations en `O(log n)`.
- Ne supporte pas `null` avec l’ordre naturel.
- Fournit l’API `NavigableSet` : `first`, `last`, `lower`, `floor`, `ceiling`, `higher`, `subSet`, `headSet`, `tailSet`…

Cas d’usage : vous avez besoin d’un ensemble trié ou d’opérations par plage.

### EnumSet

- Ensemble ultra‑optimisé pour des constantes d’une même `enum` (Utilise un bitset en interne).
- Très compact et rapide, itération dans l’ordre de déclaration des constantes.
- Ne supporte que des éléments d’une seule énumération.

Cas d’usage : flags/options, états, rôles… Exemple : `EnumSet.of(READ, WRITE)`.

### CopyOnWriteArraySet

- Thread‑safe, basé sur copie lors d’écriture (comme `CopyOnWriteArrayList`).
- Lectures très rapides et sans verrou, chaque mutation crée une nouvelle copie interne.
- À éviter si beaucoup d’écritures.

Cas d’usage : beaucoup de lectures, peu d’écritures, besoins de sécurité de thread simple.

### ConcurrentSkipListSet

- Version concurrente et triée (structure de skip‑list).
- Semantique proche d’un `TreeSet` thread‑safe, avec coûts légèrement supérieurs.

Cas d’usage : ensemble trié partagé entre threads sans verrou global.

## Complexités

- `HashSet`/`LinkedHashSet` : `add`/`contains`/`remove` ≈ `O(1)`.
- `TreeSet`/`ConcurrentSkipListSet` : `O(log n)`.
- `EnumSet` : très proche de `O(1)` pour la majorité des opérations.

## Opérations de base

```java
Set<String> tags = new HashSet<>();

boolean added = tags.add("java");     // true si l’élément n’était pas présent
added = tags.add("java");             // false (doublon ignoré)

boolean present = tags.contains("java"); // true
boolean removed = tags.remove("java");   // true
```

## Opérations ensemblistes (union, intersection, différence)

```java
Set<Integer> a = new HashSet<>(Set.of(1, 2, 3));
Set<Integer> b = new HashSet<>(Set.of(3, 4));

// union: a ∪ b => {1,2,3,4}
Set<Integer> union = new HashSet<>(a);
union.addAll(b);

// intersection: a ∩ b => {3}
Set<Integer> inter = new HashSet<>(a);
inter.retainAll(b);

// différence: a − b => {1,2}
Set<Integer> diff = new HashSet<>(a);
diff.removeAll(b);
```

## Ordre d’itération et tri

- `HashSet` : aucun ordre garanti.
- `LinkedHashSet` : ordre d’insertion (stable).
- `TreeSet` : ordre trié (naturel ou comparateur fourni).

Exemple `TreeSet` :

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

Attention : dans un `TreeSet`, deux éléments `a` et `b` sont considérés comme doublons si `compare(a, b) == 0`, même si `a.equals(b)` vaut `false`. Assurez‑vous que le comparateur est cohérent avec `equals` quand c’est important.

## Unicité : equals et hashCode

Pour `HashSet`/`LinkedHashSet`, l’unicité repose sur `hashCode` et `equals`. Quelques règles :

- Si vous redéfinissez `equals`, redéfinissez aussi `hashCode` avec une logique compatible.
- N’utilisez pas des champs mutables participant au `hashCode` si ces champs peuvent changer après insertion dans le set.

Exemple de piège :

```java
class Person {
    String ssn; // utilisé dans equals/hashCode
    String name;
    // equals/hashCode basés sur ssn
}

Set<Person> s = new HashSet<>();
Person p = new Person();
p.ssn = "123";
s.add(p);

p.ssn = "999";        // le hashCode change
boolean contains = s.contains(p); // peut retourner false !
```

Solution : rendez immuables les champs identifiants (ou n’incluez pas de champs mutables dans `equals`/`hashCode`).

## Immutabilité

- Construire un set immuable :

```java
Set<String> roles = Set.of("ADMIN", "USER"); // Java 9+
```

- Exposer une vue non modifiable d’un set interne :

```java
class Service {
    private final Set<String> scopes = new HashSet<>();
    public Set<String> getScopes() {
        return Collections.unmodifiableSet(scopes);
    }
}
```

## Déduplication et conversion

- Dédupliquer une liste :

```java
List<String> emails = List.of("a@x", "b@x", "a@x");
Set<String> unique = new LinkedHashSet<>(emails); // conserve l’ordre d’insertion
```

- Recréer une liste sans doublons (même ordre) :

```java
List<String> dedup = new ArrayList<>(new LinkedHashSet<>(emails));
```

## Concurrence

- Besoin simple de synchronisation : `Collections.synchronizedSet(new HashSet<>())`.
- Beaucoup de lectures, peu d’écritures : `CopyOnWriteArraySet`.
- Ensemble trié concurrent : `ConcurrentSkipListSet`.
- Évitez d’exposer des sets mutables partagés ; privilégiez l’immuabilité ou le confinement par thread.

## Bonnes pratiques

- Déclarez par l’interface : `Set<Foo> s = new HashSet<>();`
- Choisissez l’implémentation selon vos besoins :
  - Pas d’ordre, perfs par défaut : `HashSet`.
  - Ordre d’insertion stable : `LinkedHashSet`.
  - Ordre trié ou requêtes par plage : `TreeSet`.
  - Enumérations : `EnumSet`.
- Attention aux éléments mutables impliqués dans `equals`/`hashCode`.
- Pour un ordre d’itération stable sans coût du tri : `LinkedHashSet` plutôt que `TreeSet`.
- Utilisez `removeIf`, `addAll`, `retainAll`, `removeAll` pour exprimer des opérations ensemblistes clairement.

## Quand préférer List, Set ou Map ?

- `Set` : vous devez garantir l’unicité des éléments, l’ordre n’est pas la priorité (ou dépend de l’implémentation choisie).
- `List` : l’ordre et les doublons comptent.
- `Map` : vous avez besoin d’associer chaque clé à une valeur.

## Conclusion

Pour conclure, ce qu'il faut retenir c'est que les ensembles (Set) garantissent l’unicité des éléments, il faut cependant ne pas oublier d'implémenter `equals`/`hashCode` ou `compareTo`.

Utiliser :
- `HashSet` par défaut
- `LinkedHashSet` si l’ordre d’insertion compte
- `TreeSet` si un tri ou des bornes sont nécessaires
- `EnumSet` pour les énumérations
- Les variantes concurrentes si plusieurs threads partagent la structure.

### Pour aller plus loin

- [Set - Javadoc Java 17](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Set.html)
- [HashSet - Javadoc Java 17](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/HashSet.html)
- [LinkedHashSet - Javadoc Java 17](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/LinkedHashSet.html)
- [TreeSet/NavigableSet - Javadoc Java 17](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/TreeSet.html)
- [EnumSet - Javadoc Java 17](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/EnumSet.html)
