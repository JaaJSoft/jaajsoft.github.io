---
layout: article
title: "Virtual Threads en Java 21 : la révolution de la concurrence"
author: Pierre Chopinet
tags:
  - java
  - virtual-threads
  - concurrence
  - performance
---

Un serveur classique en Java alloue un thread par requête. Avec des threads classiques (platform threads), on atteint vite la limite : quelques milliers de threads et la JVM consomme des gigaoctets de RAM rien que pour les piles. Les Virtual Threads, introduits en Java 21, permettent de créer des millions de threads sans exploser la mémoire, en rendant la programmation concurrente aussi simple qu'un appel bloquant.
<!--more-->

Dans cet article, vous découvrirez :
- Ce que sont les Virtual Threads et comment ils fonctionnent en interne
- La différence fondamentale avec les platform threads
- Comment créer et utiliser des Virtual Threads
- Comment migrer du code existant (ExecutorService, Spring Boot)
- Les pièges à éviter : synchronisation, thread locals, CPU-bound
- L'API Structured Concurrency (preview) pour gérer les tâches concurrentes de manière structurée

Pré-requis : Java 21+ pour les Virtual Threads (JEP 444), Java 21+ preview pour Structured Concurrency (JEP 453).

---

## Le problème : un thread par requête, ça ne passe pas à l'échelle

### Le modèle classique

Dans le modèle thread-per-request, chaque requête HTTP, chaque connexion à une base de données ou chaque appel réseau bloque un thread pendant toute la durée de l'opération :

```java
// Modèle classique : un platform thread par requête
ExecutorService executor = Executors.newFixedThreadPool(200);

executor.submit(() -> {
    // Ce thread est bloqué pendant toute la durée de l'appel
    String result = httpClient.send(request, BodyHandlers.ofString()).body();
    database.save(result);  // Encore bloqué ici
    return result;
});
```

### Pourquoi ça coince

Un platform thread est un wrapper autour d'un thread OS. Chaque thread OS consomme environ **1 Mo de stack** par défaut. Les conséquences :

| Threads | RAM (stack seule) | Réalité     |
|---------|-------------------|-------------|
| 200     | ~200 Mo           | Confortable |
| 1 000   | ~1 Go             | Gérable     |
| 10 000  | ~10 Go            | Tendu       |
| 100 000 | ~100 Go           | Impossible  |

La plupart des serveurs plafonnent entre 200 et quelques milliers de threads. Le goulet d'étranglement n'est pas le CPU (qui ne fait rien pendant les I/O), mais le nombre de threads disponibles.

### Les alternatives avant Java 21

Avant les Virtual Threads, deux approches existaient :
- **I/O asynchrone** (CompletableFuture, Reactor, RxJava) : performant mais complexe, difficile à débugger, stack traces illisibles
- **Event loop** (Netty, Vert.x) : très performant mais impose un modèle de programmation radicalement différent

Les Virtual Threads offrent une troisième voie : garder le code bloquant simple, mais sans le coût mémoire des platform threads.

---

## Virtual Threads : le principe

### Qu'est-ce qu'un Virtual Thread ?

Un Virtual Thread est un thread **géré par la JVM** (et non par l'OS). Il est monté sur un platform thread (appelé **carrier thread**) uniquement quand il exécute du code. Quand il se bloque (I/O, sleep, lock), il est **démonté** du carrier, qui peut alors exécuter un autre Virtual Thread.

```
Platform Thread 1:  [VT-1 exécute] [VT-3 exécute] [VT-1 reprend] [VT-5 exécute]
Platform Thread 2:  [VT-2 exécute] [VT-4 exécute] [VT-2 reprend] [VT-6 exécute]
```

Le pool de carrier threads est dimensionné automatiquement par la JVM (par défaut, un carrier par core CPU). Un petit nombre de carriers peut servir des millions de Virtual Threads.

### Comparaison en chiffres

| Caractéristique     | Platform Thread    | Virtual Thread                                |
|---------------------|--------------------|-----------------------------------------------|
| Géré par            | OS                 | JVM                                           |
| Coût mémoire        | ~1 Mo (stack fixe) | ~quelques Ko (stack dynamique)                |
| Création            | ~1 ms              | ~1 µs                                         |
| Nombre max pratique | ~milliers          | ~millions                                     |
| Scheduling          | OS scheduler       | JVM (ForkJoinPool)                            |
| Préemption          | Oui (time-slicing) | Non (coopératif, yield aux points de blocage) |

### Ce que ça change concrètement

Avec les Virtual Threads, le code reste identique : on écrit du code bloquant classique. La différence est invisible dans le code source — elle se joue dans la JVM.

```java
// Avant (platform threads) : limité à quelques centaines de requêtes simultanées
ExecutorService executor = Executors.newFixedThreadPool(200);

// Après (virtual threads) : des millions de tâches concurrentes possibles
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
```

Une seule ligne change. Le reste du code ne bouge pas.

---

## Créer des Virtual Threads

### Avec Thread.startVirtualThread

La façon la plus directe :

```java
Thread vt = Thread.startVirtualThread(() -> {
    System.out.println("Je tourne sur un Virtual Thread : " + Thread.currentThread());
});

vt.join();  // Attendre la fin
```

### Avec Thread.ofVirtual()

Pour plus de contrôle (nom, handler d'exceptions) :

```java
Thread vt = Thread.ofVirtual()
    .name("worker-", 0)  // Nommage avec compteur : worker-0, worker-1...
    .uncaughtExceptionHandler((t, e) -> System.err.println(t.getName() + " : " + e))
    .start(() -> {
        System.out.println("Virtual Thread nommé : " + Thread.currentThread().getName());
    });
```

### Avec un ExecutorService (la méthode recommandée)

C'est l'approche à privilégier en production. `newVirtualThreadPerTaskExecutor()` crée un nouveau Virtual Thread pour chaque tâche soumise :

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    // Soumettre 10 000 tâches concurrentes
    List<Future<String>> futures = new ArrayList<>();

    for (int i = 0; i < 10_000; i++) {
        int taskId = i;
        futures.add(executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1));  // Simule un appel I/O
            return "Résultat de la tâche " + taskId;
        }));
    }

    // Récupérer les résultats
    for (Future<String> future : futures) {
        System.out.println(future.get());
    }
}
```

Ce code lance 10 000 tâches qui dorment chacune 1 seconde, et le tout se termine en ~1 seconde (pas 10 000 secondes), car les Virtual Threads bloqués ne consomment pas de carrier.

### Avec une ThreadFactory

Pour intégrer les Virtual Threads dans du code existant qui attend une `ThreadFactory` :

```java
ThreadFactory factory = Thread.ofVirtual()
    .name("vt-pool-", 0)
    .factory();

Thread t = factory.newThread(() -> System.out.println("Créé via factory"));
t.start();
```

---

## Cas pratique : serveur HTTP concurrent

Comparons un serveur qui traite des requêtes avec des platform threads puis des Virtual Threads.

### Version platform threads

```java
void startServer() throws IOException {
    var server = HttpServer.create(new InetSocketAddress(8080), 0);
    server.setExecutor(Executors.newFixedThreadPool(200));  // Max 200 requêtes simultanées

    server.createContext("/api", exchange -> {
        // Simuler un appel à une base de données (100 ms)
        Thread.sleep(100);
        byte[] response = "OK".getBytes();
        exchange.sendResponseHeaders(200, response.length);
        exchange.getResponseBody().write(response);
        exchange.close();
    });

    server.start();
}
```

Avec 200 threads, ce serveur sature à 200 requêtes concurrentes. La 201ème requête attend qu'un thread se libère.

### Version Virtual Threads

```java
void startServer() throws IOException {
    var server = HttpServer.create(new InetSocketAddress(8080), 0);
    server.setExecutor(Executors.newVirtualThreadPerTaskExecutor());  // Pas de limite pratique

    server.createContext("/api", exchange -> {
        Thread.sleep(100);
        byte[] response = "OK".getBytes();
        exchange.sendResponseHeaders(200, response.length);
        exchange.getResponseBody().write(response);
        exchange.close();
    });

    server.start();
}
```

Même code, une seule ligne modifiée. Ce serveur peut gérer des dizaines de milliers de requêtes concurrentes sans problème.

---

## Cas pratique : appels HTTP parallèles

Un cas d'usage classique : agréger les réponses de plusieurs APIs externes en parallèle.

```java
record ProductInfo(String name, double price, int stock, double rating) {}

ProductInfo fetchProductInfo(String productId) throws Exception {
    try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
        // Lancer 4 appels en parallèle
        Future<String> nameFuture = executor.submit(
            () -> callApi("/products/" + productId + "/name"));
        Future<Double> priceFuture = executor.submit(
            () -> callApi("/products/" + productId + "/price").transform(Double::parseDouble));
        Future<Integer> stockFuture = executor.submit(
            () -> callApi("/products/" + productId + "/stock").transform(Integer::parseInt));
        Future<Double> ratingFuture = executor.submit(
            () -> callApi("/products/" + productId + "/rating").transform(Double::parseDouble));

        // Attendre tous les résultats
        return new ProductInfo(
            nameFuture.get(),
            priceFuture.get(),
            stockFuture.get(),
            ratingFuture.get()
        );
    }
}
```

Avec des platform threads, ce pattern gaspillerait 4 threads du pool pour chaque appel produit. Avec des Virtual Threads, le coût est quasi nul.

---

## Migration depuis du code existant

### Remplacer un pool de threads fixe

La migration la plus simple :

```java
// Avant
ExecutorService executor = Executors.newFixedThreadPool(200);

// Après
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
```

> Le `newVirtualThreadPerTaskExecutor()` ne réutilise pas les threads : chaque tâche obtient un nouveau Virtual Thread. C'est voulu — les Virtual Threads sont tellement légers qu'un pool n'a pas de sens.

### Spring Boot

Spring Boot 3.2+ supporte les Virtual Threads nativement. Une seule propriété à activer :

```properties
# application.properties
spring.threads.virtual.enabled=true
```

Cela active les Virtual Threads pour :
- Le serveur Tomcat (chaque requête sur un Virtual Thread)
- Les tâches `@Async`
- Les listeners `@Scheduled`
- Spring WebMVC

> Les Virtual Threads ne sont **pas** utiles pour Spring WebFlux, qui utilise déjà un modèle non-bloquant.

### Ce qui fonctionne sans modification

Les Virtual Threads sont compatibles avec l'écosystème Java existant :
- `synchronized`, `ReentrantLock`, `Semaphore`
- `CompletableFuture`
- `java.net.http.HttpClient`
- JDBC (les drivers récents : PostgreSQL 42.6+, MySQL Connector 8.1+)
- La plupart des frameworks (Spring, Quarkus, Micronaut, Helidon)

---

## Les pièges à éviter

### synchronized et pinning

Quand un Virtual Thread exécute du code dans un bloc `synchronized`, il est **pinned** (épinglé) à son carrier thread. Le carrier ne peut plus servir d'autres Virtual Threads pendant ce temps.

```java
// ❌ Problème : le synchronized pin le Virtual Thread au carrier
synchronized (lock) {
    Thread.sleep(Duration.ofSeconds(1));  // Le carrier est bloqué pendant 1 seconde
}

// ✅ Solution : utiliser un ReentrantLock
private final ReentrantLock lock = new ReentrantLock();

lock.lock();
try {
    Thread.sleep(Duration.ofSeconds(1));  // Le carrier est libéré pendant le sleep
} finally {
    lock.unlock();
}
```

Pour détecter le pinning, lancez la JVM avec :

```bash
-Djdk.tracePinnedThreads=short   # Affiche un warning quand un VT est pinned
-Djdk.tracePinnedThreads=full    # Affiche la stack trace complète
```

> Le pinning n'est pas un bug, c'est une limitation technique. Il ne cause des problèmes que si le code dans le `synchronized` fait des opérations bloquantes longues.

### Thread-locals et mémoire

Les `ThreadLocal` fonctionnent avec les Virtual Threads, mais attention : avec des millions de Virtual Threads, chaque `ThreadLocal` consomme de la mémoire multipliée par le nombre de threads.

```java
// ❌ Problème : un ThreadLocal par Virtual Thread = explosion mémoire
private static final ThreadLocal<byte[]> BUFFER = ThreadLocal.withInitial(() -> new byte[1024 * 1024]);

// ✅ Solution : utiliser des Scoped Values (preview en Java 21, JEP 446)
private static final ScopedValue<RequestContext> CONTEXT = ScopedValue.newInstance();

ScopedValue.where(CONTEXT, new RequestContext(userId))
    .run(() -> {
        // CONTEXT.get() retourne le RequestContext
        processRequest();
    });
```

Les `ScopedValue` sont immuables, liées à un scope, et automatiquement nettoyées — idéales pour les Virtual Threads.

### Travail CPU-bound

Les Virtual Threads brillent pour les tâches I/O-bound (réseau, base de données, fichiers). Pour du calcul intensif (CPU-bound), ils n'apportent aucun avantage car le thread ne se bloque jamais :

```java
// ❌ Inutile : calcul CPU-bound, le Virtual Thread ne yield jamais
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> computeFibonacci(1_000_000));  // Monopolise un carrier
}

// ✅ Mieux : utiliser un pool de platform threads dimensionné au nombre de cores
ExecutorService cpuExecutor = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());
cpuExecutor.submit(() -> computeFibonacci(1_000_000));
```

### Ne pas pooler les Virtual Threads

```java
// ❌ Anti-pattern : pooler des Virtual Threads n'a aucun sens
ExecutorService pool = Executors.newFixedThreadPool(100, Thread.ofVirtual().factory());

// ✅ Correct : laisser chaque tâche avoir son propre Virtual Thread
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
```

Un pool de Virtual Threads est un contresens. Les Virtual Threads sont conçus pour être créés et jetés, pas réutilisés.

---

## FAQ

**Les Virtual Threads remplacent-ils complètement les platform threads ?**

Non. Les platform threads restent nécessaires pour les tâches CPU-bound, les threads daemon de longue durée et les cas où le contrôle précis de l'ordonnancement est requis. Les Virtual Threads sont optimisés pour les tâches I/O-bound.

**Les Virtual Threads sont-ils plus rapides ?**

Pas individuellement. Un Virtual Thread n'est pas plus rapide qu'un platform thread pour une tâche donnée. L'avantage est la **scalabilité** : vous pouvez en créer des millions là où les platform threads limitent à quelques milliers.

**Faut-il migrer tout le code existant vers les Virtual Threads ?**

Non. Migrez en priorité les chemins I/O-bound avec beaucoup de concurrence : serveurs web, clients HTTP, accès base de données. Le code CPU-bound ou le code avec peu de concurrence ne bénéficiera pas de la migration.

**Les Virtual Threads fonctionnent-ils avec les bibliothèques natives (JNI) ?**

Non, les appels JNI pinent le Virtual Thread au carrier, comme `synchronized`. Si votre code passe beaucoup de temps dans du JNI bloquant, les Virtual Threads n'apporteront pas de gain.

**Quelle version de Java minimum pour les Virtual Threads ?**

Java 21 (LTS). Les Virtual Threads étaient en preview dans Java 19 (JEP 425) et Java 20 (JEP 436), et sont devenus une feature finale dans Java 21 (JEP 444).

---

## Conclusion

Les Virtual Threads sont probablement le changement le plus impactant dans Java depuis les lambdas et les streams de Java 8. Ils résolvent un problème fondamental de scalabilité sans imposer un changement de paradigme.

**Points clés à retenir :**

- Un Virtual Thread est un thread géré par la JVM, monté/démonté dynamiquement sur un carrier thread OS
- Ils coûtent quelques Ko (vs ~1 Mo pour un platform thread) et se créent en microsecondes
- Utilisez `Executors.newVirtualThreadPerTaskExecutor()` comme point d'entrée
- Ils excellent pour les workloads I/O-bound (réseau, BDD, fichiers)
- Remplacez `synchronized` par `ReentrantLock` pour éviter le pinning
- Ne les poolez pas, ne les utilisez pas pour du calcul CPU-bound
- Activez `-Djdk.tracePinnedThreads=short` pendant le développement
- Spring Boot 3.2+ : une propriété suffit (`spring.threads.virtual.enabled=true`)

Votre code ne change quasiment pas. La JVM fait le travail.

---

## Pour aller plus loin

- [JEP 444 : Virtual Threads](https://openjdk.org/jeps/444)
- [JEP 453 : Structured Concurrency (Preview)](https://openjdk.org/jeps/453)
- [JEP 446 : Scoped Values (Preview)](https://openjdk.org/jeps/446)
- [Virtual Threads - Oracle Developer Guide](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html)

## Voir aussi

- [Comment créer ses annotations en Java]({% post_url 2026-03-09-Comment-creer-ses-annotations-en-Java %})
- [Records en Java : simplifier vos DTOs]({% post_url 2026-01-10-Records-en-Java-simplifier-vos-DTOs %})
- [Optional en Java : éviter les NullPointerException]({% post_url 2026-01-26-Optional-en-Java-eviter-les-NullPointerException %})
- [Les Sealed classes en Java]({% post_url 2026-01-14-Sealed-classes-en-Java %})
