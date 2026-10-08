---
layout: article
title: Les maps (Map) en Java
description: "Les maps en Java : interface Map, parcours, merge et computeIfAbsent, HashMap, LinkedHashMap et TreeMap, cache LRU, clés stables et ConcurrentHashMap."
author: Pierre Chopinet
tags:
  - java
  - collections
  - map
---

Cinquième partie de notre série sur les collections Java, consacrée aux maps. Une `Map` associe des clés à des valeurs, comme un annuaire associe un nom à un numéro de téléphone. Nous allons voir comment la remplir et la parcourir, les méthodes arrivées avec Java 8 (`merge`, `computeIfAbsent`...) qui évitent bien des `if`, et comment choisir entre `HashMap`, `LinkedHashMap`, `TreeMap` et les autres implémentations.
<!--more-->

1. [Introduction aux collections Java]({% post_url 2020-11-12-Framework-collections-java-intro %})
2. [Les listes (List) en Java]({% post_url 2025-09-19-Framework-collections-java-list %})
3. [Les ensembles (Set) en Java]({% post_url 2025-09-25-Framework-collections-java-set %})
4. [Les files (Queue) et Deques en Java]({% post_url 2025-09-26-Framework-collections-java-queue %})
5. Les maps (Map) en Java (vous êtes ici)
6. Utilisations avancées : [Introduction aux Streams en Java]({% post_url 2026-03-30-Introduction-aux-Streams-en-Java %}) et [Java : Comment faire des group by]({% post_url 2026-01-11-Comment-faire-des-group-by-en-Java %})

Les exemples ont été testés avec Java 21.

## L'interface Map

`Map<K, V>` n'étend pas `Collection` : c'est une famille à part, qui associe des clés de type `K` à des valeurs de type `V`. Une clé n'apparaît qu'une seule fois dans la map et n'est associée qu'à une valeur. Une même valeur peut par contre être associée à plusieurs clés.

```java
public interface Map<K, V> { /* ... */ }
```

On ajoute une association avec `put` et on la lit avec `get` :

```java
Map<String, Integer> ages = new HashMap<>();
ages.put("alice", 30);
ages.put("bob", 25);
ages.put("alice", 31);                             // remplace 30 par 31 (et retourne 30)

System.out.println(ages.get("alice"));             // 31
System.out.println(ages.get("carl"));              // null : la clé est absente
System.out.println(ages.getOrDefault("carl", 0));  // 0
System.out.println(ages.containsKey("bob"));       // true
System.out.println(ages);                          // {bob=25, alice=31}
```

Le dernier affichage montre `bob` avant `alice` : une `HashMap` ne garantit aucun ordre de parcours, ni l'ordre d'insertion ni un autre. Si l'ordre compte, `LinkedHashMap` garde l'ordre d'insertion et `TreeMap` trie les clés.

`get` retourne `null` quand la clé est absente, mais aussi quand elle est associée à la valeur `null`, que `HashMap` accepte. Dans ce cas, `getOrDefault` retourne aussi `null`, puisque la valeur par défaut ne sert que si la clé est absente : seul `containsKey` permet de faire la différence.

Les principales méthodes de l'interface (signatures simplifiées) :

| Méthode                                          | Description                                                                                                   |
|--------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| `V put(K key, V value)`                          | Associe la valeur à la clé. Retourne l'ancienne valeur, ou `null`                                             |
| `V get(Object key)`                              | Retourne la valeur associée à la clé, ou `null`                                                               |
| `V getOrDefault(Object key, V defaultValue)`     | Retourne la valeur associée à la clé, ou defaultValue si la clé est absente                                   |
| `boolean containsKey(Object key)`                | Indique si la clé est présente                                                                                |
| `boolean containsValue(Object value)`            | Indique si au moins une clé est associée à cette valeur (en parcourant toute la map)                          |
| `V remove(Object key)`                           | Supprime la clé et retourne l'ancienne valeur                                                                 |
| `boolean remove(Object key, Object value)`       | Supprime la clé seulement si elle est associée à value                                                        |
| `V replace(K key, V value)`                      | Remplace la valeur seulement si la clé est présente                                                           |
| `boolean replace(K key, V oldValue, V newValue)` | Remplace la valeur seulement si elle vaut oldValue                                                            |
| `void replaceAll(BiFunction<K, V, V> f)`         | Remplace chaque valeur par le résultat de f(clé, valeur)                                                      |
| `V putIfAbsent(K key, V value)`                  | Associe la valeur seulement si la clé est absente ou associée à `null`                                        |
| `V computeIfAbsent(K key, Function<K, V> f)`     | Si la clé est absente, calcule sa valeur avec f et l'ajoute. Retourne la valeur associée à la clé             |
| `V computeIfPresent(K key, BiFunction<K, V, V> f)` | Si la clé est présente, recalcule sa valeur avec f. Supprime la clé si f retourne `null`                    |
| `V compute(K key, BiFunction<K, V, V> f)`        | Calcule la nouvelle valeur à partir de l'ancienne (ou de `null`). Supprime la clé si f retourne `null`        |
| `V merge(K key, V value, BiFunction<V, V, V> f)` | Si la clé est absente, l'associe à value, sinon remplace la valeur par f(ancienne, value). `null` supprime la clé |
| `Set<K> keySet()`                                | Vue sur les clés                                                                                              |
| `Collection<V> values()`                         | Vue sur les valeurs                                                                                           |
| `Set<Map.Entry<K, V>> entrySet()`                | Vue sur les paires clé/valeur                                                                                 |

## Parcourir une map

Une map ne se parcourt pas directement : on parcourt l'une de ses trois vues, `keySet()` pour les clés, `values()` pour les valeurs ou `entrySet()` pour les paires clé/valeur. Quand on a besoin des deux, on parcourt `entrySet()` :

```java
Map<String, Integer> scores = new HashMap<>(Map.of("alice", 42, "bob", 8, "carl", 15));

for (Map.Entry<String, Integer> e : scores.entrySet()) {
    System.out.println(e.getKey() + " => " + e.getValue());
}
```

Ce qui donne, dans l'ordre (non garanti) de la `HashMap` :

```
bob => 8
alice => 42
carl => 15
```

`scores.forEach((k, v) -> System.out.println(k + " => " + v))` fait la même chose en une ligne. Parcourir les clés puis appeler `get` pour chacune fonctionne aussi, mais fait une recherche de plus par clé :

```java
// À éviter si on a besoin des valeurs :
for (String k : scores.keySet()) {
    Integer v = scores.get(k);
}
```

Pour modifier toutes les valeurs en place, on utilise `replaceAll`. Et comme les trois vues ne sont pas des copies mais restent reliées à la map, supprimer un élément d'une vue le supprime de la map : c'est la façon la plus simple de retirer des entrées selon une condition.

```java
scores.replaceAll((k, v) -> v == null ? 0 : v * 2);
System.out.println(scores); // {bob=16, alice=84, carl=30}

scores.values().removeIf(v -> v < 20);
System.out.println(scores); // {alice=84, carl=30}
```

Comme pour les listes, appeler `remove` sur la map pendant une boucle for-each sur l'une de ses vues lève une `ConcurrentModificationException`. Pour supprimer pendant un parcours, on utilise `removeIf` sur une vue, ou la méthode `remove()` de l'itérateur.

## merge, computeIfAbsent et les autres méthodes de Java 8

Avant Java 8, compter des mots demandait de vérifier à chaque fois si le mot était déjà dans la map :

```java
Integer n = freqs.get(mot);
if (n == null) {
    freqs.put(mot, 1);
} else {
    freqs.put(mot, n + 1);
}
```

`merge` fait la même chose en une ligne. Si la clé est absente, elle l'associe à la valeur donnée, sinon elle remplace la valeur actuelle par le résultat de la fonction, appliquée à l'ancienne valeur et à la nouvelle :

```java
String texte = "Le chat voit le chien, le chien dort.";

Map<String, Integer> freqs = new HashMap<>();
for (String mot : texte.split("\\W+")) {
    if (mot.isEmpty()) continue;
    freqs.merge(mot.toLowerCase(), 1, Integer::sum);
}
System.out.println(freqs); // {dort=1, voit=1, chat=1, le=3, chien=2}
```

Attention avec un texte en français : par défaut, `\W` considère les lettres accentuées comme des séparateurs, et `"L'été arrive à grands pas".split("\\W+")` donne `[L, t, arrive, grands, pas]`. Le préfixe `(?U)`, qui active les classes de caractères Unicode, règle le problème : `split("(?U)\\W+")` donne bien `[L, été, arrive, à, grands, pas]`.

`computeIfAbsent` calcule et ajoute la valeur seulement si la clé est absente, et retourne dans tous les cas la valeur associée à la clé. On peut donc enchaîner directement un `add` pour remplir une map de listes :

```java
List<String> noms = List.of("alice", "bob", "Anna", "claire", "Bruno");

Map<Character, List<String>> index = new HashMap<>();
for (String nom : noms) {
    char k = Character.toUpperCase(nom.charAt(0));
    index.computeIfAbsent(k, key -> new ArrayList<>()).add(nom);
}
System.out.println(index); // {A=[alice, Anna], B=[bob, Bruno], C=[claire]}
```

C'est un regroupement par clé, que les Streams savent aussi faire avec `Collectors.groupingBy` (voir [les group by en Java]({% post_url 2026-01-11-Comment-faire-des-group-by-en-Java %})).

Les autres méthodes suivent la même logique : `putIfAbsent` n'ajoute la valeur que si la clé est absente, `computeIfPresent` ne recalcule la valeur que si la clé est présente, et `compute` la recalcule dans tous les cas. Pour `compute`, `computeIfPresent` et `merge`, une fonction qui retourne `null` supprime la clé de la map.

## Les implémentations

| Implémentation      | Ordre de parcours                   | Clé `null`                   | `get`, `put`, `remove` |
|---------------------|-------------------------------------|------------------------------|------------------------|
| `HashMap`           | Aucun ordre garanti                 | Acceptée                     | O(1)                   |
| `LinkedHashMap`     | Ordre d'insertion (ou d'accès)      | Acceptée                     | O(1)                   |
| `TreeMap`           | Trié par clé                        | Refusée avec l'ordre naturel | O(log n)               |
| `EnumMap`           | Ordre de déclaration des constantes | Refusée                      | O(1)                   |
| `ConcurrentHashMap` | Aucun ordre garanti                 | Refusée, comme les valeurs `null` | O(1)              |
| `Hashtable`         | Aucun ordre garanti                 | Refusée, comme les valeurs `null` | O(1)              |

`HashMap` est le choix par défaut. Elle range ses entrées dans un tableau de cases selon le `hashCode` de leur clé, et `get`, `put` et `remove` se font en temps constant, à condition que les `hashCode` des clés soient bien répartis. Depuis Java 8, une case qui accumule trop de collisions (8 entrées) est transformée en arbre rouge-noir, ce qui limite la casse quand la fonction de hachage est mauvaise.

Quand le nombre d'entrées dépasse 75 % du nombre de cases (le facteur de charge par défaut, 0,75), la table double de taille et toutes les entrées sont redistribuées. Si vous connaissez à l'avance le nombre d'entrées, autant créer la map à la bonne taille, mais attention : le paramètre de `new HashMap<>(n)` est un nombre de cases, pas un nombre d'entrées. Pour 100 entrées, `new HashMap<>(100)` crée une table de 128 cases, qui passe à 256 cases au 97e ajout. Depuis Java 19, `HashMap.newHashMap(100)` calcule la bonne capacité à partir du nombre d'entrées attendu.

`LinkedHashMap` ajoute à la table de hachage une liste doublement chaînée qui relie les entrées dans leur ordre d'insertion. Remettre une clé déjà présente avec `put` ne change pas sa place. Elle peut aussi suivre l'ordre d'accès, ce qui permet d'écrire un cache LRU en quelques lignes.

`TreeMap` garde ses entrées triées par clé dans un arbre rouge-noir, selon l'ordre naturel des clés ou selon le `Comparator` passé au constructeur. Ses opérations se font en O(log n), et elle permet de chercher par plage de clés.

`EnumMap` est réservée aux clés d'un même `enum`. C'est un simple tableau indexé par la position de la constante dans l'enum, plus compact et en général plus rapide qu'une `HashMap`. L'article sur [les enums en Java]({% post_url 2026-04-13-Les-Enums-en-Java-bien-plus-que-des-constantes %}) montre comment l'utiliser.

Deux implémentations plus spécialisées complètent la liste. `WeakHashMap` ne retient pas ses clés : quand une clé n'est plus référencée ailleurs dans le programme, le ramasse-miettes peut la libérer et son entrée disparaît de la map, ce qui permet d'associer des informations à des objets sans les empêcher d'être libérés. `IdentityHashMap` compare les clés avec `==` au lieu de `equals` : deux objets égaux mais distincts y sont deux clés différentes. Elle enfreint volontairement le contrat de `Map` et ne sert que dans des cas précis, comme la copie d'un graphe d'objets, où il faut retenir les objets déjà traités.

`ConcurrentHashMap` et `Hashtable`, faites pour les programmes multithreads, sont présentées plus bas avec les threads.

## Un cache LRU avec LinkedHashMap

Un cache LRU (*Least Recently Used*) a une taille limitée, et quand il est plein, il supprime l'entrée qui n'a pas servi depuis le plus longtemps. `LinkedHashMap` fait presque tout le travail. Le troisième paramètre de son constructeur, `accessOrder`, lui fait suivre l'ordre d'accès plutôt que l'ordre d'insertion : chaque lecture ou écriture d'une entrée la place en dernière position. Et sa méthode `removeEldestEntry`, appelée après chaque ajout, supprime la première entrée (la plus ancienne) quand elle retourne `true` :

```java
class LruCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxEntries;

    LruCache(int maxEntries) {
        super(16, 0.75f, true); // accessOrder = true
        this.maxEntries = maxEntries;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > maxEntries;
    }
}
```

Avec un cache de deux entrées :

```java
Map<String, String> cache = new LruCache<>(2);
cache.put("a", "A");
cache.put("b", "B");
cache.get("a");                     // "a" devient l'entrée utilisée le plus récemment
cache.put("c", "C");                // trois entrées : "b", la moins récemment utilisée, est supprimée
System.out.println(cache.keySet()); // [a, c]
```

## Rechercher par plage avec TreeMap

`TreeMap` implémente `NavigableMap`, qui permet de chercher par rapport à une clé : `firstKey()` et `lastKey()`, `lowerEntry()`, `floorKey()`, `ceilingEntry()` et les autres variantes pour la clé juste avant ou juste après une valeur, `headMap()`, `tailMap()` et `subMap()` pour une plage de clés.

```java
NavigableMap<Integer, String> m = new TreeMap<>();
m.put(10, "dix");
m.put(20, "vingt");
m.put(30, "trente");
System.out.println(m.subMap(10, true, 20, true)); // {10=dix, 20=vingt}
```

`floorEntry` est pratique pour classer une valeur dans des tranches. Ici, chaque clé est la note minimale d'une mention :

```java
NavigableMap<Integer, String> mentions = new TreeMap<>();
mentions.put(0, "insuffisant");
mentions.put(10, "passable");
mentions.put(12, "assez bien");
mentions.put(14, "bien");
mentions.put(16, "très bien");

System.out.println(mentions.floorEntry(13).getValue()); // assez bien
```

`floorEntry(13)` retourne l'entrée dont la clé est la plus grande clé inférieure ou égale à 13, ici 12.

## Les clés doivent être stables

Les règles vues pour les éléments d'un `HashSet` dans [la partie sur les ensembles]({% post_url 2025-09-25-Framework-collections-java-set %}) s'appliquent aux clés d'une `HashMap` (un `HashSet` est d'ailleurs une `HashMap` dont on n'utilise que les clés). Les méthodes `equals` et `hashCode` des clés doivent être cohérentes entre elles, et une clé ne doit pas être modifiée tant qu'elle est dans la map : si son `hashCode` change, la map ne la retrouve plus. Les meilleures clés sont des objets immuables, comme `String`, `Integer`, `UUID` ou les records.

Attention aussi aux types numériques. `get` accepte n'importe quel `Object`, si bien que le code suivant compile, mais un `Integer` n'est jamais égal à un `Long`, même s'ils ont la même valeur :

```java
Map<Long, String> clients = new HashMap<>();
clients.put(42L, "Alice");
System.out.println(clients.get(42));   // null : 42 est un Integer, pas un Long
System.out.println(clients.get(42L));  // Alice
```

## Maps et threads

`HashMap`, `LinkedHashMap` et `TreeMap` ne sont pas synchronisées. Pour partager une map entre plusieurs threads, il faut utiliser `ConcurrentHashMap`. Ses lectures ne prennent pas de verrou, ses écritures ne bloquent pas toute la table, et ses méthodes `putIfAbsent`, `compute`, `computeIfAbsent`, `computeIfPresent` et `merge` sont atomiques : deux threads qui font un `merge` sur la même clé au même moment ne perdent pas de mise à jour. Ce n'est pas le cas des implémentations par défaut de ces méthodes dans l'interface `Map`, qui ne garantissent rien en cas d'accès concurrents.

`ConcurrentHashMap` refuse les clés et les valeurs `null` : un `get` qui retourne `null` signifie donc toujours que la clé est absente, sans qu'on ait besoin d'appeler `containsKey`, qu'un autre thread pourrait de toute façon contredire entre les deux appels.

`Collections.synchronizedMap(new HashMap<>())` synchronise chaque méthode d'une map ordinaire, avec la même limite que `synchronizedList` : les parcours doivent se faire dans un bloc `synchronized` sur la map. Quant à `Hashtable`, présente depuis Java 1.0, c'est l'ancêtre synchronisé de `HashMap`. Sa documentation recommande d'utiliser `HashMap` à sa place si l'on n'a pas besoin de synchronisation, et `ConcurrentHashMap` sinon.

## Maps non modifiables

Comme pour les listes et les ensembles, Java 9 a ajouté des fabriques qui créent des maps non modifiables :

```java
// Map.of : jusqu'à 10 paires clé/valeur
Map<String, Integer> ages = Map.of(
    "alice", 30,
    "bob",   25
);
// ages est immuable : toute tentative de modification lève UnsupportedOperationException

// Map.ofEntries : pratique au-delà de 10 entrées ou pour plus de lisibilité
Map<String, Integer> scores = Map.ofEntries(
    Map.entry("A", 1),
    Map.entry("B", 2),
    Map.entry("C", 3)
);

// À partir d'une Map existante, créer une copie immuable :
Map<String, Integer> unmodifiable = Map.copyOf(scores);
```

Ces trois méthodes refusent les clés et les valeurs `null` (`NullPointerException`), et `Map.of` et `Map.ofEntries` refusent les clés en double (`IllegalArgumentException: duplicate key`). La map elle-même ne peut pas changer, mais si ses valeurs sont des objets modifiables, une liste par exemple, rien n'empêche de modifier ces objets.

Leur ordre de parcours n'est pas spécifié, et en pratique il change d'une exécution à l'autre : `System.out.println(Map.of("alice", 30, "bob", 25, "carl", 41))` affiche tantôt `{carl=41, alice=30, bob=25}`, tantôt `{bob=25, alice=30, carl=41}`. Pour un ordre fixe, construisez plutôt une `LinkedHashMap` avec des `put` successifs.

Pour exposer une map interne en lecture seule sans la copier, `Collections.unmodifiableMap(map)` retourne une vue non modifiable, qui suit les changements de la map d'origine.

Voilà pour la dernière famille de collections. La série continue avec les utilisations avancées : les [Streams]({% post_url 2026-03-30-Introduction-aux-Streams-en-Java %}), qui permettent de traiter les collections sans écrire de boucles, et les [group by en Java]({% post_url 2026-01-11-Comment-faire-des-group-by-en-Java %}), qui produisent justement des maps.

## Voir aussi

- [Les ensembles (Set) en Java]({% post_url 2025-09-25-Framework-collections-java-set %})
- [Les enums en Java]({% post_url 2026-04-13-Les-Enums-en-Java-bien-plus-que-des-constantes %})
- [Records en Java : simplifier vos DTOs]({% post_url 2026-01-10-Records-en-Java-simplifier-vos-DTOs %})
- [Javadoc de Map (Java 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Map.html)
- [Javadoc de HashMap (Java 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/HashMap.html)
- [Javadoc de LinkedHashMap (Java 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/LinkedHashMap.html)
- [Javadoc de TreeMap (Java 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/TreeMap.html)
- [Javadoc de ConcurrentHashMap (Java 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html)
