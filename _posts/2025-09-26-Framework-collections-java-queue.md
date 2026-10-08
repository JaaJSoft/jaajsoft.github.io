---
layout: article
title: Les files (Queue) et Deques en Java
description: "Les files en Java : Queue et Deque, ArrayDeque pour les files et les piles, PriorityQueue, et les files bloquantes pour faire travailler des threads ensemble."
author: Pierre Chopinet
tags:
  - java
  - collections
  - queue
  - deque
---

Quatrième partie de notre série sur les collections Java : les files. Une `Queue` stocke des éléments en attendant de les traiter, dans leur ordre d'arrivée ou par priorité, et son extension `Deque` permet d'ajouter et de retirer des éléments aux deux bouts. Nous allons voir leurs méthodes, les implémentations à utiliser au quotidien (`ArrayDeque`, `PriorityQueue`), puis les files bloquantes qui servent à faire travailler des threads ensemble.
<!--more-->

1. [Introduction aux collections Java]({% post_url 2020-11-12-Framework-collections-java-intro %})
2. [Les listes (List) en Java]({% post_url 2025-09-19-Framework-collections-java-list %})
3. [Les ensembles (Set) en Java]({% post_url 2025-09-25-Framework-collections-java-set %})
4. Les files (Queue) et Deques en Java (vous êtes ici)
5. [Les maps (Map) en Java]({% post_url 2025-10-04-Framework-collections-java-map %})
6. Utilisations avancées : [Introduction aux Streams en Java]({% post_url 2026-03-30-Introduction-aux-Streams-en-Java %}) et [Java : Comment faire des group by]({% post_url 2026-01-11-Comment-faire-des-group-by-en-Java %})

Les exemples ont été testés avec Java 21.

## L'interface Queue

`Queue` est une sous-interface de `Collection` faite pour garder des éléments en attente de traitement. Le plus souvent, l'ordre est FIFO (*first in, first out*) : on ajoute les éléments en fin de file et on retire le plus ancien, en tête de file. Ce n'est pas une obligation, une file à priorité par exemple fait sortir les éléments selon leur priorité. Il n'y a pas d'accès par index : on ne travaille qu'avec la tête de la file.

```java
public interface Queue<E> extends Collection<E> { /* ... */ }
```

Chaque opération existe en deux versions : l'une lève une exception quand elle ne peut pas aboutir, l'autre retourne une valeur spéciale (`false` ou `null`).

| Opération                    | Version qui lève une exception                              | Version qui retourne une valeur spéciale  |
|------------------------------|-------------------------------------------------------------|-------------------------------------------|
| Ajouter en fin de file       | `add(e)` : `IllegalStateException` si la file est pleine    | `offer(e)` : `false` si la file est pleine |
| Retirer la tête              | `remove()` : `NoSuchElementException` si la file est vide   | `poll()` : `null` si la file est vide     |
| Consulter la tête sans la retirer | `element()` : `NoSuchElementException` si la file est vide | `peek()` : `null` si la file est vide     |

La plupart des files n'ont pas de limite de taille, et l'ajout ne peut tout simplement pas échouer. `offer` est prévu pour les files à capacité bornée, où une file pleine est une situation normale et pas une erreur. Pour retirer ou consulter la tête, tout dépend de ce que signifie une file vide dans votre code : si elle ne devrait jamais l'être, `remove()` et `element()` le signalent par une exception, sinon `poll()` et `peek()` évitent un `try`/`catch`.

C'est aussi pour cette raison que la plupart des implémentations refusent les éléments `null` : comme `poll()` et `peek()` retournent `null` quand la file est vide, on ne pourrait pas faire la différence avec un élément `null`. `LinkedList` les accepte, mais la documentation de `Queue` déconseille d'en ajouter, même dans ce cas.

## Deque, une file à deux bouts

`Deque` (pour *double ended queue*, qui se prononce "deck") étend `Queue` pour ajouter, retirer et consulter des éléments aux deux extrémités. Depuis Java 21, elle étend aussi `SequencedCollection`, comme les listes.

```java
public interface Deque<E> extends Queue<E>, SequencedCollection<E> { /* ... */ }
```

Chaque opération de `Queue` y existe pour chacun des deux bouts, toujours avec une version qui lève une exception et une version qui retourne une valeur spéciale :

|           | Début de la deque               | Fin de la deque               |
|-----------|---------------------------------|-------------------------------|
| Ajouter   | `addFirst(e)`, `offerFirst(e)`  | `addLast(e)`, `offerLast(e)`  |
| Retirer   | `removeFirst()`, `pollFirst()`  | `removeLast()`, `pollLast()`  |
| Consulter | `getFirst()`, `peekFirst()`     | `getLast()`, `peekLast()`     |

Les méthodes héritées de `Queue` font de la `Deque` une file FIFO : `offer` ajoute à la fin, `poll` retire au début. Elle peut aussi servir de pile LIFO (*last in, first out*) avec `push(e)`, l'équivalent de `addFirst(e)`, et `pop()`, l'équivalent de `removeFirst()`. C'est d'ailleurs ce que recommande la documentation de la vieille classe `Stack`, présente depuis Java 1.0 : utiliser une `Deque` à sa place.

## ArrayDeque, pour les files et les piles

`ArrayDeque` est l'implémentation de `Deque` à utiliser par défaut. Elle range ses éléments dans un tableau circulaire qui s'agrandit au besoin, et les ajouts et retraits aux deux extrémités se font en temps constant amorti. Sa documentation précise qu'elle est probablement plus rapide que `Stack` pour faire une pile, et que `LinkedList` pour faire une file. Elle refuse les éléments `null` et n'est pas thread-safe.

La même `ArrayDeque` peut servir de file, en ajoutant à la fin et en retirant au début, ou de pile, avec `push` et `pop` qui travaillent au début :

```java
Deque<String> dq = new ArrayDeque<>();
dq.addLast("A");  // enqueue
dq.addLast("B");
String head = dq.removeFirst(); // "A"
dq.push("X");                 // pile => [X, B]
String top = dq.pop();          // "X"
```

`LinkedList` implémente elle aussi `Deque`, en plus de `List`. Elle accepte les `null`, mais chaque élément occupe un nœud en mémoire : elle n'a d'intérêt que si vous avez besoin d'un objet qui soit à la fois une `Deque` et une `List`.

L'exemple classique de file FIFO est le parcours en largeur d'un graphe : on visite d'abord les voisins directs du nœud de départ, puis les voisins de ces voisins, et ainsi de suite.

```java
void bfs(Node start) {
    Set<Node> visited = new HashSet<>();
    Deque<Node> q = new ArrayDeque<>();
    q.add(start);
    visited.add(start);
    while (!q.isEmpty()) {
        Node n = q.removeFirst();
        for (Node nb : n.neighbors()) {
            if (visited.add(nb)) {
                q.addLast(nb);
            }
        }
    }
}
```

Chaque nœud retiré de la tête de la file y fait entrer ses voisins à la fin. `visited.add(nb)` retourne `false` si le nœud a déjà été vu, ce qui évite de le remettre dans la file et de tourner en rond quand le graphe contient des cycles.

## PriorityQueue, la file à priorité

`PriorityQueue` ne fait pas sortir les éléments dans leur ordre d'arrivée, mais du plus petit au plus grand, selon leur ordre naturel ou selon le `Comparator` passé au constructeur. Elle s'appuie sur un tas binaire : `offer` et `poll` se font en O(log n) et `peek` en temps constant, mais `contains` et `remove(Object)` en O(n). Elle refuse les `null`.

```java
Queue<Integer> pq = new PriorityQueue<>();
pq.offer(5);
pq.offer(1);
pq.offer(3);
System.out.println(pq.peek()); // 1
System.out.println(pq);        // [1, 5, 3]

while (!pq.isEmpty()) {
    System.out.print(pq.poll() + " "); // 1 3 5
}
```

Comme le montre le `println`, seule la tête de la file est garantie d'être le plus petit élément : le reste du tas n'est que partiellement trié, et un parcours avec une boucle for-each ne suit aucun ordre particulier. Pour récupérer les éléments dans l'ordre, il faut les retirer un par un avec `poll()`, comme ici.

Pour faire sortir d'abord le plus grand élément, on passe `Comparator.reverseOrder()` au constructeur. Attention aussi aux ex aequo : quand plusieurs éléments ont la même priorité, la documentation précise que l'ordre entre eux est arbitraire. Une `PriorityQueue` ne respecte donc pas l'ordre d'arrivée des éléments de même priorité.

C'est la structure qu'on utilise pour traiter des tâches par ordre d'urgence, ou dans l'algorithme de Dijkstra pour choisir à chaque étape le nœud le plus proche du point de départ.

## Les files bloquantes pour faire travailler des threads ensemble

`ArrayDeque` et `PriorityQueue` ne sont pas thread-safe. Le package `java.util.concurrent` propose des files faites pour être partagées entre threads, en particulier les `BlockingQueue`, dont certaines méthodes savent attendre : `put(e)` attend qu'il y ait de la place dans la file, `take()` attend qu'un élément arrive, et `offer(e, délai, unité)` et `poll(délai, unité)` attendent au plus le temps indiqué.

Les principales implémentations sont :

- `ArrayBlockingQueue`, une file bornée basée sur un tableau dont la taille est fixée à la création. Une option du constructeur garantit aux threads en attente d'être servis dans leur ordre d'arrivée.
- `LinkedBlockingQueue`, basée sur des nœuds chaînés, bornée si on lui donne une capacité, sinon limitée à `Integer.MAX_VALUE` éléments.
- `PriorityBlockingQueue`, l'équivalent bloquant de `PriorityQueue`, sans limite de taille.
- `DelayQueue`, une file d'éléments `Delayed` qui ne peuvent être retirés qu'une fois leur délai écoulé.
- `SynchronousQueue`, une file sans aucune capacité : chaque `put` attend qu'un autre thread fasse un `take`, l'élément passe directement de l'un à l'autre.
- `LinkedBlockingDeque`, la version `Deque` de `LinkedBlockingQueue`.

En dehors des `BlockingQueue`, `ConcurrentLinkedQueue` est une file FIFO non bornée et non bloquante : ses opérations ne prennent pas de verrou, et `poll()` retourne `null` tout de suite si la file est vide. Attention, sa méthode `size()` n'est pas en temps constant, elle doit parcourir toute la file.

L'utilisation typique d'une file bloquante est le modèle producteur/consommateur : un thread produit des tâches, un autre les traite. Une file bornée régule le débit : si le consommateur prend du retard, le producteur reste bloqué sur `put` au lieu d'accumuler des tâches. Avec une file non bornée, un producteur durablement plus rapide que son consommateur ferait grossir la file jusqu'à épuiser la mémoire.

```java
BlockingQueue<String> q = new ArrayBlockingQueue<>(100);

Thread producer = new Thread(() -> {
    try {
        for (int i = 0; i < 10_000; i++) {
            q.put("job-" + i); // bloque si plein
        }
        q.put("FIN");          // prévient le consommateur qu'il n'y a plus rien à traiter
    } catch (InterruptedException ie) { Thread.currentThread().interrupt(); }
});

Thread consumer = new Thread(() -> {
    try {
        while (true) {
            String job = q.take(); // bloque si vide
            if (job.equals("FIN")) {
                break;
            }
            process(job);
        }
    } catch (InterruptedException ie) { Thread.currentThread().interrupt(); }
});

producer.start();
consumer.start();
```

Une `BlockingQueue` n'a pas de méthode pour signaler qu'il n'y aura plus d'éléments, et elle refuse les `null`. Pour arrêter le consommateur, le producteur envoie donc une valeur spéciale, ici `"FIN"`, que la documentation appelle un objet *poison*. Sans elle, le consommateur attendrait indéfiniment sur `take()` et le programme ne se terminerait jamais.

Quand on ne veut pas attendre indéfiniment, `poll` accepte un délai maximal et retourne `null` s'il n'a rien reçu à temps :

```java
BlockingQueue<Task> q = new LinkedBlockingQueue<>(1000);
try {
    Task t = q.poll(100, TimeUnit.MILLISECONDS); // null si timeout
} catch (InterruptedException ie) {
    Thread.currentThread().interrupt();
}
```

La dernière famille de collections, celle des [maps]({% post_url 2025-10-04-Framework-collections-java-map %}), associe des clés à des valeurs : c'est le sujet de la partie suivante.

## Voir aussi

- [Les listes (List) en Java]({% post_url 2025-09-19-Framework-collections-java-list %})
- [Les Virtual Threads en Java 21]({% post_url 2026-03-23-Virtual-Threads-en-Java-21 %})
- [Javadoc de Queue (Java 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Queue.html)
- [Javadoc de Deque (Java 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Deque.html)
- [Javadoc d'ArrayDeque (Java 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/ArrayDeque.html)
- [Javadoc de PriorityQueue (Java 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/PriorityQueue.html)
- [Javadoc de BlockingQueue (Java 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/BlockingQueue.html)
