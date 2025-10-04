---
layout: article
title: Les files (Queue) en Java
author: Pierre Chopinet
tags:
  - java
  - collections
  - queue
  - deque
---

Dans cet article (partie 4 de la série sur les collections), nous allons nous concentrer sur la famille Queue du Framework Collections, ainsi que sur son extension Deque. Nous verrons leurs caractéristiques, les principales implémentations (ArrayDeque, LinkedList, PriorityQueue, BlockingQueue/Deque…), leurs différences, pièges courants et bonnes pratiques d’utilisation.
<!--more-->

1. [Introduction aux collections Java]({% post_url 2020-11-12-Framework-collections-java-intro %})
2. [Les listes en Java]({% post_url 2025-09-19-Framework-collections-java-list %})
3. [Les ensembles (Set) en Java]({% post_url 2025-09-25-Framework-collections-java-set %})
4. Les files (Queue) et Deques en Java (vous êtes ici)
5. [Les maps (Map) en Java]({% post_url 2025-10-04-Framework-collections-java-map %})
6. Utilisations avancées des collections (article à venir)

## Qu’est‑ce qu’une Queue ?

`Queue` est une sous‑interface de `Collection` conçue pour gérer des éléments dans un ordre d’extraction défini, typiquement FIFO (First-In, First-Out).

Signature :

```java
public interface Queue<E> extends Collection<E> { /* … */ }
```

Principes :
- Ordre d’extraction souvent FIFO (sauf exceptions comme `PriorityQueue`).
- Pas d’accès par index, on manipule la tête (head) et parfois la queue (tail).
- Certaines implémentations peuvent avoir une capacité bornée (ex. `ArrayBlockingQueue`).

## Méthodes de l’interface Queue

| Méthode            | Échec si vide/plein | Description                                                                         |
|--------------------|---------------------|-------------------------------------------------------------------------------------|
| boolean add(E e)   | Exception           | Enfile (enqueue) l’élément, échoue avec `IllegalStateException` si capacité pleine. |
| boolean offer(E e) | false               | Enfile sans exception, retourne false si pas de place.                              |
| E remove()         | Exception           | Défile (dequeue) et retourne la tête, `NoSuchElementException` si vide.             |
| E poll()           | null                | Défile et retourne la tête, `null` si vide.                                         |
| E element()        | Exception           | Consulte la tête sans la retirer, exception si vide.                                |
| E peek()           | null                | Consulte la tête, `null` si vide.                                                   |

Remarques :
- Préférez `offer`/`poll`/`peek` dans le code robuste, pour éviter la gestion d’exceptions de contrôle.
- `Queue` n’autorise pas `null` dans la plupart des implémentations (car `null` sert de valeur spéciale pour `poll`/`peek`).

## Qu’est‑ce qu’une Deque ?

`Deque` (Double Ended Queue) étend `Queue` pour permettre des opérations en tête et en queue, supportant aussi les usages de pile (LIFO).

Signature :

```java
public interface Deque<E> extends Queue<E> { /* … */ }
```

Méthodes principales (équivalents tête/queue) :

| Ajout                 | Retrait                | Consultation           |
|-----------------------|------------------------|------------------------|
| addFirst, offerFirst  | removeFirst, pollFirst | getFirst, peekFirst    |
| addLast, offerLast    | removeLast, pollLast   | getLast, peekLast      |

Aliases utiles :
- `push(e)` ≡ `addFirst(e)` (usage pile)
- `pop()` ≡ `removeFirst()`

## Principales implémentations

### ArrayDeque

- Structure : tableau circulaire redimensionnable (pas de capacité fixe, mais redimensionnements amortis).
- Opérations en tête/queue en `O(1)`.
- N’autorise pas `null`.
- Souvent plus rapide que `LinkedList` pour `Deque`, et préférable à l’ancienne `Stack`.

Cas d’usage : file FIFO, pile LIFO, parcours en largeur (BFS), buffers temporaires.

Exemple :

```java
Deque<String> dq = new ArrayDeque<>();
dq.addLast("A");  // enqueue
dq.addLast("B");
String head = dq.removeFirst(); // "A"
dq.push("X");                 // pile => [X, B]
String top = dq.pop();          // "X"
```

### LinkedList (implémente List et Deque)

- Liste doublement chaînée.
- Fournit toutes les opérations `Deque` mais avec un surcoût mémoire et un cache CPU moins favorable.
- Accès par index coûteux (`O(n)`), mais insertion/retrait aux extrémités efficaces.

Cas d’usage : utile si vous avez absolument besoin d’une `Deque` qui soit aussi une `List`, sinon préférez `ArrayDeque`.

### PriorityQueue (file à priorité)

- Structure : tas binaire (min-heap par défaut).
- L’élément en tête est le « plus petit » selon l’ordre naturel ou un `Comparator` fourni.
- Pas FIFO : l’ordre dépend des priorités.
- N’autorise pas `null`.
- Iterator non trié (ne pas s’y fier pour l’ordre). Pour itérer trié : dépilez via `poll()` successifs.

Cas d’usage : planification par priorité, Dijkstra, k plus petits éléments, scheduling simple.

Exemple :

```java
Queue<Integer> pq = new PriorityQueue<>();
pq.offer(5);
pq.offer(1);
pq.offer(3);
// head -> 1
while (!pq.isEmpty()) {
    System.out.print(pq.poll() + " "); // 1 3 5
}
```

### Files concurrentes et bloquantes (java.util.concurrent)

- `ConcurrentLinkedQueue` : non bloquante (lock‑free), FIFO, non bornée. Excellente pour la haute concurrence en lecture/écriture.
- `ArrayBlockingQueue` : bornée, basée sur tableau, opérations bloquantes (`put`, `take`), option d’équité.
- `LinkedBlockingQueue` : (optionnellement) bornée, basée sur liens.
- `PriorityBlockingQueue` : comme `PriorityQueue` mais thread‑safe (non bornée).
- `DelayQueue` : éléments disponibles après un délai (`Delayed`).
- `SynchronousQueue` : capacité zéro (handoff direct producteur→consommateur).
- `LinkedBlockingDeque` : version `Deque` bloquante.

Exemple producteur/consommateur :

```java
BlockingQueue<String> q = new ArrayBlockingQueue<>(100);

Thread producer = new Thread(() -> {
    try {
        for (int i = 0; i < 10_000; i++) {
            q.put("job-" + i); // bloque si plein
        }
    } catch (InterruptedException ie) { Thread.currentThread().interrupt(); }
});

Thread consumer = new Thread(() -> {
    try {
        while (true) {
            String job = q.take(); // bloque si vide
            process(job);
        }
    } catch (InterruptedException ie) { Thread.currentThread().interrupt(); }
});

producer.start();
consumer.start();
```

## Pièges courants

- `null` rarement autorisé : préférez des sentinelles explicites ou `Optional` côté API, pas d’éléments `null` dans les queues.
- `remove` vs `poll` : `remove` lève une exception si la file est vide, `poll` retourne `null`.
- `PriorityQueue` : ne garantit pas un ordre trié à l’itération, seulement pour la tête. Utiliser `poll()` pour dépiler en ordre.
- `Comparator` incohérent : avec `PriorityQueue`, un comparateur non transitif ou non cohérent peut produire des comportements surprenants.
- Capacité non bornée : les queues non bornées peuvent entraîner des fuites de mémoire côté producteur/consommateur. Préférez des files bornées pour réguler le débit.

## Bonnes pratiques

- Déclarez par l’interface : `Queue<T>`, `Deque<T>`.
- Pour une queue/pile en mémoire : préférez `ArrayDeque`.
- Évitez `Stack` (héritage historique) : utilisez `Deque` (`ArrayDeque`) pour les piles LIFO.
- Choisissez entre `offer`/`poll`/`peek` (retours spéciaux) et `add`/`remove`/`element` (exceptions) selon votre style d’erreur.
- En concurrence : utilisez les `BlockingQueue` pour backpressure (le producteur/consommateur attend quand la file est pleine/vide) et simplicité, ou `ConcurrentLinkedQueue` pour non bloquant.
- Documentez la politique de priorité si vous exposez une `PriorityQueue` dans une API.

## Exemples utiles

### Parcours en largeur (BFS) avec ArrayDeque

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

### Utiliser Deque comme pile

```java
Deque<Integer> stack = new ArrayDeque<>();
stack.push(10);
stack.push(20);
int top = stack.pop(); // 20
```

### Timeout avec BlockingQueue

```java
BlockingQueue<Task> q = new LinkedBlockingQueue<>(1000);
Task t = q.poll(100, TimeUnit.MILLISECONDS); // null si timeout
```

## Quand préférer Queue/Deque, List ou Set ?

- Utilisez `Queue`/`Deque` pour des modèles FIFO/LIFO, pour traiter des éléments dans un certain ordre d’arrivée ou de priorité, et pour des opérations efficaces en tête/queue.
- Utilisez `List` si l’ordre indexé compte et que les doublons sont permis.
- Utilisez `Set` si vous devez garantir l’unicité des éléments.

## Conclusion

`Queue` et `Deque` complètent `List` et `Set` en fournissant des structures adaptées au traitement en flux et aux scénarios de producteur/consommateur, avec des implémentations efficaces (ArrayDeque) et des variantes prioritaires et concurrentes.
Bien choisir l’implémentation et les méthodes utilisées (`offer`/`poll`/`peek` vs `add`/`remove`/`element`) va rendre votre code plus robuste et performant.

Pour aller plus loin dans la série :

1. [Introduction aux collections Java]({% post_url 2020-11-12-Framework-collections-java-intro %})
2. [Les listes en Java]({% post_url 2025-09-19-Framework-collections-java-list %})
3. [Les ensembles (Set) en Java]({% post_url 2025-09-25-Framework-collections-java-set %})
4. Les files (Queue) et Deques en Java (vous êtes ici)
5. [Les maps (Map) en Java]({% post_url 2025-10-04-Framework-collections-java-map %})
6. Utilisations avancées des collections (article à venir)

### Pour aller plus loin

- [Queue - Javadoc Java 17](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Queue.html)
- [Deque - Javadoc Java 17](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Deque.html)
- [ArrayDeque - Javadoc Java 17](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/ArrayDeque.html)
- [PriorityQueue - Javadoc Java 17](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/PriorityQueue.html)
- [java.util.concurrent (BlockingQueue…) - Javadoc Java 17](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/package-summary.html)
