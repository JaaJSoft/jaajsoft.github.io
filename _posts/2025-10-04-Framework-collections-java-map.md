---
layout: article
title: Les maps (Map) en Java
author: Pierre Chopinet
tags:
  - java
  - collections
  - map
---

Dans cet article (partie 5 de la série sur les collections), nous allons nous concentrer sur la famille Map du Framework Collections. Nous verrons ses principes (association clé/valeur), les principales implémentations (HashMap, LinkedHashMap, TreeMap, etc), leurs différences, pièges courants et bonnes pratiques d’utilisation.
<!--more-->

1. [Introduction aux collections Java]({% post_url 2020-11-12-Framework-collections-java-intro %})
2. [Les listes (List) en Java]({% post_url 2025-09-19-Framework-collections-java-list %})
3. [Les ensembles (Set) en Java]({% post_url 2025-09-25-Framework-collections-java-set %})
4. [Les files (Queue) et Deques en Java]({% post_url 2025-09-26-Framework-collections-java-queue %})
5. Les maps (Map) en Java (vous êtes ici)
6. Utilisations avancées des collections (article à venir)

## Qu’est‑ce qu’une Map ?

`Map<K, V>` n’étend pas `Collection` : c’est une famille à part qui gère des associations clé/valeur. Chaque clé est unique au sein de la map et pointe vers au plus une valeur.

Signature :

```java
public interface Map<K, V> { /* … */ }
```

Principes :
- Unicité des clés (la notion d’égalité dépend de l’implémentation : equals/hashCode, ordre trié, identité…).
- Une clé peut être associée à null selon l’implémentation.
- La plupart des implémentations ne sont pas thread‑safe par défaut (sauf `ConcurrentHashMap`, `Collections.synchronizedMap`, etc.).

## Méthodes clés de Map

| Méthode                                        | Description                                                                           |
|------------------------------------------------|---------------------------------------------------------------------------------------|
| V put(K key, V value)                          | Ajoute/remplace la valeur de key. Retourne l’ancienne valeur ou null.                 |
| V get(Object key)                              | Récupère la valeur associée, ou null si absente.                                      |
| V getOrDefault(Object key, V defaultValue)     | Récupère la valeur, ou defaultValue si absente.                                       |
| boolean containsKey(Object key)                | true si la clé est présente.                                                          |
| boolean containsValue(Object value)            | true si au moins une entrée a cette valeur.                                           |
| V remove(Object key)                           | Supprime la clé et retourne l’ancienne valeur.                                        |
| boolean remove(Object key, Object value)       | Supprime seulement si la clé est associée à value.                                    |
| V replace(K key, V value)                      | Remplace seulement si présente. Retourne l’ancienne valeur ou null.                   |
| boolean replace(K key, V oldVal, V newVal)     | Remplace conditionnellement.                                                          |
| void replaceAll(BiFunction<K,V,V> f)           | Remplace chaque valeur par f.apply(k, v).                                             |
| V putIfAbsent(K key, V value)                  | Met la valeur seulement si absente (ou associée à null selon implémentation).         |
| V compute(K key, BiFunction<K,V,V> remap)      | Recalcule la valeur à partir de l’ancienne. Supprime si remap retourne null.          |
| V computeIfAbsent(K key, Function<K,V> m)      | Calcule et insère seulement si absente.                                               |
| V computeIfPresent(K key, BiFunction<K,V,V> m) | Recalcule seulement si présente. Supprime si m retourne null.                         |
| V merge(K key, V value, BiFunction<V,V,V> f)   | Si absente, put(value). Sinon, remplace par f.apply(old, value). null => suppression. |
| Set<K> keySet()                                | Vue des clés.                                                                         |
| Collection<V> values()                         | Vue des valeurs.                                                                      |
| Set<Map.Entry<K,V>> entrySet()                 | Vue des entrées (clé/valeur).                                                         |

Remarques :
- Les vues `keySet`, `values`, `entrySet` sont liées à la map : modifier la vue modifie la map (et inversement).
- Les opérations de type `compute*` et `merge` sont très utiles pour éviter les if/put répétitifs et sont atomiques sur `ConcurrentHashMap`.

### Parcourir une Map

- Parcourir les paires clé/valeur (le plus courant) :

```java
for (Map.Entry<String, Integer> e : scores.entrySet()) {
    System.out.println(e.getKey() + " => " + e.getValue());
}
```

- Parcourir seulement les clés :

```java
for (String k : scores.keySet()) {
    Integer v = scores.get(k);
}
```

- Remplacer des valeurs en place :

```java
scores.replaceAll((k, v) -> v == null ? 0 : v * 2);
```

- Compter les fréquences avec `merge` :

```java
for (String mot : mots) {
    freqs.merge(mot, 1, Integer::sum);
}
```

## Principales implémentations

### HashMap

- Structure : table de hachage (buckets + arbres rouges/noirs au‑delà d’un seuil de collisions depuis Java 8).
- Ordre d’itération non garanti et susceptible de changer.
- Opérations de base (`get`, `put`, `remove`) en O(1).
- Accepte une clé null et des valeurs null (plusieurs).
- Paramètres importants : capacité initiale, facteur de charge (load factor, 0.75 par défaut).

Cas d’usage : choix par défaut pour une map non ordonnée.

### LinkedHashMap

- Même base que HashMap + liste doublement chaînée pour l’ordre d’insertion ou d’accès.
- Ordre d’itération prévisible ; peut servir de LRU simple avec `accessOrder=true` et `removeEldestEntry`.

Exemple LRU 100 entrées :

```java
Map<K, V> cache = new LinkedHashMap<>(16, 0.75f, true) {
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return size() > 100;
    }
};
```

### TreeMap (SortedMap/NavigableMap)

- Structure : arbre rouge‑noir, tri selon l’ordre naturel ou un `Comparator` passé au constructeur.
- Opérations en O(log n), itération triée.
- Ne supporte pas les clés null avec l’ordre naturel.
- API Navigable : `firstKey`, `lastKey`, `lowerEntry`, `floorKey`, `ceilingEntry`, `subMap`, `headMap`, `tailMap`…

Cas d’usage : besoin de tri, de recherches par plage, de bornes.

### Hashtable (legacy)

- Ancienne map synchronisée, à éviter au profit de `ConcurrentHashMap` ou `Collections.synchronizedMap`.
- N’accepte pas null (ni clé ni valeur).

### ConcurrentHashMap

- Map concurrente haute performance.
- Pas de null (ni clé ni valeur) pour éviter des ambiguïtés (`get` ne peut pas retourner null pour « absent » vs « valeur null »).
- Opérations atomiques utiles : `compute*`, `merge`, `putIfAbsent`.
- Bonne évolutivité sous contention (segmentation interne + verrous fins/cas sans verrou selon opérations).

Cas d’usage : partage de données entre threads avec très peu de blocages.

### WeakHashMap

- Les clés sont faiblement référencées : si une clé n’est plus référencée ailleurs, l’entrée peut être collectée par le GC.
- Typiquement utilisé pour des caches de métadonnées non essentiels.

### IdentityHashMap

- L’égalité des clés est basée sur l’identité (`==`) et non `equals`.
- Très spécialisé (frameworks, graphes d’objets, déduplication par identité).

### EnumMap

- Map optimisée pour des clés d’un même type `enum`.
- Très compacte, itération dans l’ordre de déclaration des constantes.
- N’accepte pas null.

## Égalité, hashCode et clés correctes

- Pour les maps basées sur le hachage (`HashMap`, `LinkedHashMap`), les clés doivent respecter le contrat `equals`/`hashCode` : deux clés égales (equals==true) doivent avoir le même hashCode.
- Clés immuables recommandées (String, Integer, UUID, objets valeur immuables). Évitez les clés mutables insérées puis modifiées, qui « perdront » leur case de hachage.
- Attention aux types numériques différents (Integer vs Long) : `equals` retournera false, même si la valeur apparente est la même.

## Nulls, ordre et vues

- `HashMap`/`LinkedHashMap` acceptent une clé null et des valeurs null.
- `TreeMap` avec ordre naturel n’accepte pas de clé null.
- `ConcurrentHashMap` n’accepte aucun null.
- Les vues `keySet`, `values`, `entrySet` sont dynamiques : supprimer via `iterator.remove()` sur `entrySet` est le moyen sûr pour retirer en cours d’itération.

## Complexités et performances

- `HashMap` : `get`/`put`/`remove` ≈ O(1) ; pires cas O(n) mais mitigés par l’arborisation des buckets en cas de nombreuses collisions.
- `LinkedHashMap` : proche de `HashMap` + léger surcoût d’ordre.
- `TreeMap` : `get`/`put`/`remove` ≈ O(log n) ; itération triée.
- `ConcurrentHashMap` : très bonne scalabilité, opérations atomiques utiles.

## Bonnes pratiques

- Choisir la bonne clé : immuable, avec des `equals`/`hashCode` corrects.
- Définir une capacité initiale si vous connaissez la taille cible pour limiter les réallocations.
- Préférer `getOrDefault`, `computeIfAbsent`, `merge` aux séquences if/contains/put fragiles.
- Ne pas itérer avec `keySet` si vous avez besoin des valeurs : utilisez `entrySet()`.
- Pour un cache LRU simple, `LinkedHashMap` + `accessOrder=true` + `removeEldestEntry`.
- En concurrence : privilégier `ConcurrentHashMap`. Évitez `Hashtable`.

## Pièges courants

- Modifier une clé après insertion (mutable) : accès et suppression deviennent impossibles via la clé modifiée.
- Confondre `containsKey` et `containsValue` (souvent, on veut `containsKey`).
- Supposer un ordre stable avec `HashMap` : non garanti, ne basez pas votre logique dessus.
- Utiliser null avec `ConcurrentHashMap` : interdit.
- Oublier que `entrySet`, `keySet`, `values` sont des vues : certaines opérations modifient directement la map.

## Exemples

### Comptage de mots avec merge

```java
Map<String, Integer> freqs = new HashMap<>();
for (String mot : texte.split("\\W+")) {
    if (mot.isEmpty()) continue;
    freqs.merge(mot.toLowerCase(), 1, Integer::sum);
}
```

### Index inversé (clé = lettre initiale)

```java
Map<Character, List<String>> index = new HashMap<>();
for (String nom : noms) {
    char k = Character.toUpperCase(nom.charAt(0));
    index.computeIfAbsent(k, key -> new ArrayList<>()).add(nom);
}
```

### TreeMap et recherches par plage

```java
NavigableMap<Integer, String> m = new TreeMap<>();
m.put(10, "dix");
m.put(20, "vingt");
m.put(30, "trente");
System.out.println(m.subMap(10, true, 20, true)); // {10=dix, 20=vingt}
```

### Map immuable avec Map.of / Map.ofEntries

```java
// Map.of: jusqu'à 10 paires clé/valeur
Map<String, Integer> ages = Map.of(
    "alice", 30,
    "bob",   25
);
// ages est immuable: toute tentative de modification lève UnsupportedOperationException

// Map.ofEntries: pratique au‑delà de 10 entrées ou pour plus de lisibilité
Map<String, Integer> scores = Map.ofEntries(
    Map.entry("A", 1),
    Map.entry("B", 2),
    Map.entry("C", 3)
);

// À partir d’une Map existante, créer une copie immuable:
Map<String, Integer> unmodifiable = Map.copyOf(scores);
```

Notes:
- Map.of / Map.ofEntries / Map.copyOf rejettent les clés ou valeurs null (NullPointerException).
- Map.of rejette les clés dupliquées (IllegalArgumentException).
- Les maps retournées sont non modifiables; si les valeurs sont mutables, elles peuvent toujours changer indépendamment.

## Conclusion

`Map` est la brique clé pour modéliser des associations clé/valeur en Java. En choisissant l’implémentation adaptée (rapidité vs ordre vs tri vs concurrence) et en appliquant de bonnes pratiques (clés immuables, API compute/merge), vous éviterez la plupart des pièges et écrirez un code plus clair et plus performant.

Pour aller plus loin dans la série :

1. [Introduction aux collections Java]({% post_url 2020-11-12-Framework-collections-java-intro %})
2. [Les listes (List) en Java]({% post_url 2025-09-19-Framework-collections-java-list %})
3. [Les ensembles (Set) en Java]({% post_url 2025-09-25-Framework-collections-java-set %})
4. [Les files (Queue) et Deques en Java]({% post_url 2025-09-26-Framework-collections-java-queue %})
5. Les maps (Map) en Java (vous êtes ici)
6. Utilisations avancées des collections (article à venir)

### Pour aller plus loin

- [Map - Javadoc Java 17](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Map.html)
- [HashMap - Javadoc Java 17](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/HashMap.html)
- [LinkedHashMap - Javadoc Java 17](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/LinkedHashMap.html)
- [TreeMap/NavigableMap - Javadoc Java 17](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/TreeMap.html)
- [ConcurrentHashMap - Javadoc Java 17](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html)
- [WeakHashMap - Javadoc Java 17](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/WeakHashMap.html)
- [IdentityHashMap - Javadoc Java 17](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/IdentityHashMap.html)
- [EnumMap - Javadoc Java 17](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/EnumMap.html)
