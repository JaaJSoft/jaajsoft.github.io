---
layout: article
title: Les listes en Java - comprendre leur fonctionnement, quand les utiliser et lesquelles utiliser
excerpt:
author: Pierre Chopinet
tags:
  - java
  - collections
  - list
---

Dans cet article (partie 2 de la série sur les collections), nous allons nous concentrer sur la famille List du Framework Collections : ses caractéristiques, ses principales implémentations (ArrayList, LinkedList…), leurs performances et les bonnes pratiques d’utilisation au quotidien.
<!--more-->

1. [Introduction aux collections Java]({% post_url 2020-11-12-Framework-collections-java-intro %})
2. Les listes en Java (vous êtes ici)
3. Collections Set (article à venir)
4. Collections Queue (article à venir)
5. Collections Map (article à venir)
6. Utilisations avancées des collections (article à venir)

## Qu’est‑ce qu’une List ?

`List` est une sous‑interface de `Collection` qui représente une séquence ordonnée d’éléments :

- Les éléments ont un ordre (position/index).
- Les doublons sont autorisés.
- L’accès indexé est possible (méthodes `get`, `set`, `add(int, E)`, `remove(int)`, etc.).

La signature :

```java
public interface List<E> extends Collection<E> { /* … */ }
```

En plus des méthodes héritées de `Collection`, `List` ajoute entre autres :

- `E get(int index)`, `E set(int index, E element)`
- `void add(int index, E element)`, `E remove(int index)`
- `int indexOf(Object o)`, `int lastIndexOf(Object o)`
- `ListIterator<E> listIterator()`, `ListIterator<E> listIterator(int index)`
- `List<E> subList(int fromIndex, int toIndex)` (vue de la liste)

## Principales implémentations

### ArrayList

- Structure interne : tableau redimensionnable.
- Accès par index très rapide (en `O(1)`), insertion/suppression en fin rapides, mais insertion/suppression au milieu coûteuses (décalage des éléments).
- Mémoire compacte, très utilisée par défaut.

Cas d’usage : lecture fréquente par index, ajout en fin, listes majoritairement immuables après construction.

### LinkedList

- Liste doublement chaînée (chaque élément connaît son précédent et son suivant).
- Insertion/suppression au début/au milieu rapides si vous disposez déjà d’un `ListIterator` positionné, mais accès par index en `O(n)`.
- Surcoût mémoire (pointeurs) + moins bonne avec le cache CPU.

Cas d’usage : scénarios spécialisés où l’on enlève/insère souvent au milieu via un itérateur, ou quand on utilise l’API `Deque` que `LinkedList` implémente aussi (pile/queue simple).

### Autres options utiles

- `CopyOnWriteArrayList` : thread‑safe, excellente en lecture majoritaire (chaque écriture copie la liste). À éviter si beaucoup d’écritures.
- `Collections.synchronizedList(new ArrayList<>())` : wrapper synchronisé basique.
- Listes immuables : `List.of(...)` (Java 9+) ou `Collections.unmodifiableList(list)` pour exposer une vue non modifiable.

## Complexités et performances (rappels)

- Accès indexé : `ArrayList` ≈ `O(1)` amorti, `LinkedList` ≈ `O(n)`.
- Ajout en fin : `ArrayList` ≈ `O(1)` amorti, `LinkedList` ≈ `O(1)` (avec pointeur de fin).
- Insertion/suppression au milieu par index : `ArrayList` ≈ `O(n)`, `LinkedList` ≈ `O(n)` (traversée) mais `O(1)` si itérateur déjà positionné.

En pratique, `ArrayList` est souvent le meilleur choix par défaut, `LinkedList` gagnant seulement dans des cas de niches avec itérateur.

## Méthodes clés à connaître

### Ajout, lecture, mise à jour

```java
List<String> fruits = new ArrayList<>();
fruits.add("pomme");           // [pomme]
fruits.add("banane");          // [pomme, banane]
fruits.add(1, "poire");        // [pomme, poire, banane]

String second = fruits.get(1);  // "poire"
fruits.set(2, "prune");        // [pomme, poire, prune]
```

### Suppression par index ou par valeur

Attention à la surcharge `remove(int)` vs `remove(Object)` :

```java
fruits.remove(1);               // supprime l’élément à l’index 1
fruits.remove("pomme");        // supprime la première occurrence de "pomme"
```

### Recherche et sous‑liste

```java
int i = fruits.indexOf("prune");     // -1 si absent
List<String> debut = fruits.subList(0, 2); // vue [0,2[
```

`subList` retourne une vue adossée à la liste d’origine : modifier l’une modifie l’autre. Pour obtenir une copie indépendante :

```java
List<String> copie = new ArrayList<>(fruits.subList(0, 2));
```

### Tri, remplacement, filtrage

```java
fruits.sort(Comparator.naturalOrder());     // tri croissant
fruits.replaceAll(String::toUpperCase);     // applique la fonction à chaque élément
fruits.removeIf(s -> s.length() <= 4);      // filtre en place
```

## Itérateurs et parcours

- Préférez le `for-each` pour la lecture :

```java
for (String f : fruits) {
    System.out.println(f);
}
```

- Pour supprimer en parcourant, utilisez l’itérateur (ou `removeIf`) pour éviter `ConcurrentModificationException` :

```java
Iterator<String> it = fruits.iterator();
while (it.hasNext()) {
    if (it.next().startsWith("P")) {
        it.remove(); // OK
    }
}
```

## ListIterator : parcours bidirectionnel

`ListIterator` permet de parcourir dans les deux sens et d’insérer/remplacer au vol :

```java
List<Integer> nums = new ArrayList<>(List.of(1, 2, 3));
ListIterator<Integer> li = nums.listIterator();
while (li.hasNext()) {
    Integer n = li.next();
    if (n == 2) {
        li.add(99);   // insère avant l’élément suivant
    }
}
// nums => [1, 2, 99, 3]
```

## Immutabilité et exposition sûre d’API

- Construire directement une liste immuable :

```java
List<String> roles = List.of("ADMIN", "USER");
```

- Exposer une vue non modifiable d’une liste interne :

```java
class Service {
    private final List<String> logs = new ArrayList<>();
    public List<String> getLogs() {
        return Collections.unmodifiableList(logs);
    }
}
```

## Concurrence

- Lecture majoritaire, peu d’écritures : `CopyOnWriteArrayList`.
- Besoin simple de synchronisation : `Collections.synchronizedList(new ArrayList<>())`.
- Sinon, envisagez une conception sans partage (immutabilité, confinement de thread) ou des structures concurrentes adaptées.

## Bonnes pratiques

- Déclarez par l’interface : `List<String> l = new ArrayList<>();`
- Choisissez l’implémentation selon le pattern d’accès (lecture par index ? insertions fréquentes ?).
- Évitez `LinkedList` par défaut ; mesurez avant d’optimiser.
- Attention à `remove` en boucle for‑each : utilisez un itérateur ou `removeIf`.
- Copiez une `subList` si vous avez besoin d’une liste indépendante.

## Quand préférer une List, un Set ou une Queue ?

- Utilisez `List` si l’ordre compte et si les doublons sont permis.
- Utilisez `Set` si vous devez garantir l’unicité des éléments.
- Utilisez `Queue`/`Deque` pour des modèles FIFO/LIFO et des opérations en tête/queue optimisées.

## Conclusion

`List` est probablement la collection la plus utilisée en Java. En comprenant ses implémentations clés, leurs complexités et les pièges courants, vous ferez des choix plus éclairés et écrirez un code plus robuste et performant.

### Pour aller plus loin

- https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/List.html
- https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/ArrayList.html
- https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/LinkedList.html
