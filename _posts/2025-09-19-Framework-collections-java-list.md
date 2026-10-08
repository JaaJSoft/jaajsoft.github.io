---
layout: article
title: Les listes (List) en Java
description: "Les listes en Java : interface List, ArrayList ou LinkedList, sous-listes, tri, parcours et ListIterator, listes non modifiables et accès concurrents."
author: Pierre Chopinet
tags:
  - java
  - collections
  - list
---

Deuxième partie de notre série sur les collections Java : après la présentation générale du Framework Collections, nous passons aux listes. Nous allons voir ce que l'interface `List` ajoute à `Collection`, comment choisir entre `ArrayList` et `LinkedList`, et les quelques pièges de l'API (`remove(int)` contre `remove(Object)`, les sous-listes, la suppression pendant un parcours).
<!--more-->

1. [Introduction aux collections Java]({% post_url 2020-11-12-Framework-collections-java-intro %})
2. Les listes (List) en Java (vous êtes ici)
3. [Les ensembles (Set) en Java]({% post_url 2025-09-25-Framework-collections-java-set %})
4. [Les files (Queue) et Deques en Java]({% post_url 2025-09-26-Framework-collections-java-queue %})
5. [Les maps (Map) en Java]({% post_url 2025-10-04-Framework-collections-java-map %})
6. Utilisations avancées : [Introduction aux Streams en Java]({% post_url 2026-03-30-Introduction-aux-Streams-en-Java %}) et [Java : Comment faire des group by]({% post_url 2026-01-11-Comment-faire-des-group-by-en-Java %})

Les exemples ont été testés avec Java 21.

## L'interface List

Une liste est une séquence ordonnée d'éléments : chaque élément a une position, son index (qui commence à 0), et un même élément peut apparaître plusieurs fois. C'est ce qui la différencie d'un ensemble (`Set`), qui refuse les doublons et ne connaît pas la notion d'index.

```java
public interface List<E> extends SequencedCollection<E> { /* ... */ }
```

Jusqu'à Java 20, `List` héritait directement de `Collection`. Java 21 a intercalé entre les deux l'interface `SequencedCollection`, commune à toutes les collections dont les éléments ont un ordre défini. Elle apporte `getFirst()`, `getLast()`, `addFirst()`, `addLast()`, `removeFirst()`, `removeLast()` et `reversed()`.

En plus des méthodes de `Collection` vues dans la première partie, `List` propose des méthodes qui travaillent avec les positions :

| Méthode                                       | Description                                                                 |
|-----------------------------------------------|-----------------------------------------------------------------------------|
| `E get(int index)`                            | Retourne l'élément à l'index donné                                          |
| `E set(int index, E element)`                 | Remplace l'élément à l'index donné et retourne l'ancien                     |
| `void add(int index, E element)`              | Insère l'élément à l'index donné, en décalant les suivants                  |
| `E remove(int index)`                         | Supprime l'élément à l'index donné et le retourne                           |
| `int indexOf(Object o)`                       | Retourne l'index de la première occurrence de o, ou -1 s'il est absent      |
| `int lastIndexOf(Object o)`                   | Retourne l'index de la dernière occurrence de o, ou -1 s'il est absent      |
| `ListIterator<E> listIterator()`              | Retourne un itérateur capable de parcourir la liste dans les deux sens      |
| `ListIterator<E> listIterator(int index)`     | Même chose, en démarrant à l'index donné                                    |
| `List<E> subList(int fromIndex, int toIndex)` | Retourne une vue sur les éléments entre fromIndex (inclus) et toIndex (exclu) |
| `void replaceAll(UnaryOperator<E> operator)`  | Remplace chaque élément par le résultat de la fonction                      |
| `void sort(Comparator<? super E> c)`          | Trie la liste selon le comparateur                                          |

## ArrayList ou LinkedList ?

`ArrayList` range ses éléments dans un tableau. L'accès à un élément par son index est donc immédiat, quelle que soit la taille de la liste. Quand le tableau est plein, la liste en alloue un plus grand et y recopie ses éléments : certains ajouts coûtent cher, mais un ajout en fin de liste reste en temps constant en moyenne (on parle de temps constant amorti). Par contre, insérer ou supprimer un élément au début ou au milieu oblige à décaler tous les éléments qui suivent.

`LinkedList` est une liste doublement chaînée : chaque élément est rangé dans un nœud qui garde une référence vers le nœud précédent et vers le suivant. Ajouter ou retirer un élément à une extrémité est immédiat, et au milieu aussi, à condition d'y être déjà positionné avec un itérateur. En revanche, pour atteindre l'élément d'index `i`, la liste doit suivre les nœuds un par un, en partant de l'extrémité la plus proche.

| Opération                                    | `ArrayList` | `LinkedList` |
|----------------------------------------------|-------------|--------------|
| `get(i)`, `set(i, e)`                        | O(1)        | O(n)         |
| `add(e)` (en fin de liste)                   | O(1) amorti | O(1)         |
| `add(0, e)`, `remove(0)`                     | O(n)        | O(1)         |
| `add(i, e)`, `remove(i)`                     | O(n)        | O(n)         |
| `add` ou `remove` à travers un `ListIterator` | O(n)        | O(1)         |
| `contains(o)`, `indexOf(o)`                  | O(n)        | O(n)         |

Pour `add(i, e)` et `remove(i)`, les deux listes sont en O(n), mais pas pour la même raison : `ArrayList` doit décaler les éléments, `LinkedList` doit parcourir ses nœuds jusqu'à l'index.

Dans la grande majorité des cas, `ArrayList` est le bon choix. À complexité égale, elle va plus vite qu'une `LinkedList` (sa documentation le précise), et chaque nœud d'une `LinkedList` occupe bien plus de mémoire qu'une case de tableau. `LinkedList` ne se justifie que si l'on insère et supprime beaucoup en tête de liste ou à travers un itérateur, et même dans ce cas, mieux vaut mesurer avant de changer. Et pour une file, `ArrayDeque` (que nous verrons dans la partie 4) sera probablement plus rapide qu'une `LinkedList`.

## Ajouter, lire et modifier des éléments

On ajoute en fin de liste avec `add`, ou à une position donnée avec `add(index, element)`. `get` lit l'élément à un index et `set` le remplace :

```java
List<String> fruits = new ArrayList<>();
fruits.add("pomme");           // [pomme]
fruits.add("banane");          // [pomme, banane]
fruits.add(1, "poire");        // [pomme, poire, banane]

String second = fruits.get(1); // "poire"
fruits.set(2, "prune");        // [pomme, poire, prune]
```

`indexOf` et `lastIndexOf` retournent la position de la première et de la dernière occurrence d'un élément, ou -1 s'il est absent. Comme `contains`, elles parcourent la liste en comparant les éléments avec `equals`.

```java
int i = fruits.indexOf("prune");  // 2
int j = fruits.indexOf("fraise"); // -1
```

Depuis Java 21, plus besoin d'écrire `fruits.get(fruits.size() - 1)` pour lire le dernier élément :

```java
System.out.println(fruits.getFirst()); // pomme
System.out.println(fruits.getLast());  // prune
System.out.println(fruits.reversed()); // [prune, poire, pomme]
```

`reversed()` ne copie pas la liste : elle retourne une vue qui la présente dans l'ordre inverse.

Attention à `remove`, qui existe en deux versions : `remove(int index)` supprime l'élément à une position, `remove(Object o)` supprime la première occurrence d'un élément. Avec une liste de chaînes, il n'y a pas d'ambiguïté. Avec une `List<Integer>`, par contre, passer un entier appelle la version avec l'index, car le compilateur préfère la méthode qui ne demande pas de conversion en `Integer` :

```java
List<Integer> nombres = new ArrayList<>(List.of(10, 20, 1, 30));
nombres.remove(1);                  // supprime l'élément d'index 1, donc 20
System.out.println(nombres);        // [10, 1, 30]

nombres.remove(Integer.valueOf(1)); // supprime la valeur 1
System.out.println(nombres);        // [10, 30]
```

## Les sous-listes

`subList(from, to)` retourne la portion de liste comprise entre l'index `from` inclus et l'index `to` exclu. Ce n'est pas une copie mais une vue sur la liste d'origine : ce qu'on modifie à travers la sous-liste est répercuté sur la liste. On peut s'en servir pour supprimer une plage d'éléments en une ligne :

```java
List<Integer> valeurs = new ArrayList<>(List.of(1, 2, 3, 4, 5, 6));
valeurs.subList(1, 4).clear();
System.out.println(valeurs); // [1, 5, 6]
```

Dans l'autre sens, ça se passe mal : si la liste d'origine change de taille sans passer par la sous-liste, le comportement de la sous-liste n'est plus défini. Avec une `ArrayList`, le prochain accès à la sous-liste lève une exception :

```java
List<Integer> debut = valeurs.subList(0, 2);
valeurs.add(7);
System.out.println(debut); // java.util.ConcurrentModificationException
```

Pour garder une portion de liste indépendante de l'originale, on en fait une copie :

```java
List<String> copie = new ArrayList<>(fruits.subList(0, 2)); // [pomme, poire]
```

## Trier et transformer une liste

Trois méthodes modifient la liste en place : `sort` la trie selon un comparateur, `replaceAll` remplace chaque élément par le résultat d'une fonction, et `removeIf` supprime les éléments qui vérifient une condition.

```java
List<String> panier = new ArrayList<>(List.of("prune", "kiwi", "pomme", "figue", "poire"));

panier.sort(Comparator.naturalOrder());
System.out.println(panier); // [figue, kiwi, poire, pomme, prune]

panier.replaceAll(String::toUpperCase);
System.out.println(panier); // [FIGUE, KIWI, POIRE, POMME, PRUNE]

panier.removeIf(s -> s.length() <= 4);
System.out.println(panier); // [FIGUE, POIRE, POMME, PRUNE]
```

Pour obtenir une nouvelle liste en laissant l'originale intacte, on passera plutôt par un Stream, comme expliqué dans l'[introduction aux Streams]({% post_url 2026-03-30-Introduction-aux-Streams-en-Java %}).

## Parcourir une liste

Pour lire les éléments un par un, la boucle for-each suffit :

```java
List<String> fruits = new ArrayList<>(List.of("pomme", "kiwi", "poire", "figue"));

for (String f : fruits) {
    System.out.println(f);
}
```

Par contre, il ne faut pas supprimer d'élément de la liste dans cette boucle. Elle utilise un itérateur, qui détecte que la liste a été modifiée sans passer par lui et lève une `ConcurrentModificationException` :

```java
// À éviter :
for (String f : fruits) {
    if (f.startsWith("p")) {
        fruits.remove(f); // ConcurrentModificationException au tour suivant
    }
}
```

Et l'exception n'est même pas garantie. Si l'élément supprimé est l'avant-dernier de la liste, la boucle s'arrête sans erreur, mais sans avoir vu le dernier élément : avec la liste `[kiwi, pomme, poire]`, la boucle supprime `pomme`, s'arrête, et `poire` reste dans la liste.

Pour supprimer pendant un parcours, il faut passer par la méthode `remove()` de l'itérateur, ou plus simplement utiliser `removeIf` :

```java
// Correct :
Iterator<String> it = fruits.iterator();
while (it.hasNext()) {
    if (it.next().startsWith("p")) {
        it.remove();
    }
}

// Ou en une ligne :
fruits.removeIf(f -> f.startsWith("p"));
```

Dans les deux cas, il reste `[kiwi, figue]`.

`ListIterator`, obtenu avec `listIterator()`, va plus loin qu'un `Iterator` classique : il parcourt la liste dans les deux sens (`hasPrevious()` et `previous()`) et peut remplacer (`set`) ou insérer (`add`) des éléments en cours de route. `add` insère le nouvel élément juste avant celui que retournerait le prochain appel à `next()` :

```java
List<Integer> nums = new ArrayList<>(List.of(1, 2, 3));
ListIterator<Integer> li = nums.listIterator();
while (li.hasNext()) {
    Integer n = li.next();
    if (n == 2) {
        li.add(99);   // insère avant l'élément suivant
    }
}
System.out.println(nums); // [1, 2, 99, 3]
```

Pour parcourir la liste à l'envers, on part de la fin avec `listIterator(nums.size())` et on remonte avec `previous()`.

## Listes non modifiables

`List.of(...)`, disponible depuis Java 9, crée une liste non modifiable : toute tentative d'ajout, de suppression ou de remplacement lève une `UnsupportedOperationException`. Elle refuse aussi les éléments `null`.

```java
List<String> roles = List.of("ADMIN", "USER");
roles.add("GUEST"); // java.lang.UnsupportedOperationException
```

Pour exposer une liste interne à une classe sans laisser l'appelant la modifier, `Collections.unmodifiableList` retourne une vue en lecture seule :

```java
class Service {
    private final List<String> logs = new ArrayList<>();

    public List<String> getLogs() {
        return Collections.unmodifiableList(logs);
    }
}
```

Comme c'est une vue, l'appelant verra apparaître les éléments que le service ajoutera plus tard dans `logs`. S'il lui faut une copie figée, on retourne plutôt `List.copyOf(logs)` (Java 10+), qui est elle aussi non modifiable.

## Listes et threads

`ArrayList` et `LinkedList` ne sont pas synchronisées : si plusieurs threads modifient la même liste, c'est à vous de gérer la synchronisation. La bibliothèque standard propose deux solutions toutes prêtes.

`Collections.synchronizedList(new ArrayList<>())` retourne une liste dont toutes les méthodes sont synchronisées. Attention, cela ne couvre pas les parcours : pendant une boucle, un autre thread peut toujours modifier la liste. La documentation demande donc de faire les parcours dans un bloc `synchronized` sur la liste :

```java
List<String> liste = Collections.synchronizedList(new ArrayList<>());

synchronized (liste) {
    for (String s : liste) {
        System.out.println(s);
    }
}
```

`CopyOnWriteArrayList`, du package `java.util.concurrent`, prend le problème à l'envers : chaque modification recopie entièrement le tableau interne, et chaque parcours travaille sur le tableau tel qu'il était au moment où le parcours a commencé. Les lectures ne prennent aucun verrou et un parcours ne lève jamais de `ConcurrentModificationException`. C'est intéressant pour une liste très souvent lue et rarement modifiée, mais à éviter si les écritures sont fréquentes, puisque chacune copie toute la liste.

Dans la partie suivante, nous verrons les [ensembles]({% post_url 2025-09-25-Framework-collections-java-set %}), pour les cas où les doublons n'ont pas leur place.

## Voir aussi

- [Les files (Queue) et Deques en Java]({% post_url 2025-09-26-Framework-collections-java-queue %})
- [Introduction aux Streams en Java]({% post_url 2026-03-30-Introduction-aux-Streams-en-Java %})
- [Javadoc de List (Java 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/List.html)
- [Javadoc d'ArrayList (Java 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/ArrayList.html)
- [Javadoc de LinkedList (Java 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/LinkedList.html)
