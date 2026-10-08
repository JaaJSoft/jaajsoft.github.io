---
layout: article
title: "Les Virtual Threads en Java 21"
description: "Les virtual threads de Java 21 : fonctionnement, création, serveur HTTP, appels en parallèle, migration depuis un pool de threads et pinning avec synchronized."
author: Pierre Chopinet
tags:
  - java
  - virtual-threads
  - concurrence
  - performance
---

Dans un serveur Java classique, chaque requête occupe un thread du début à la fin, y compris pendant qu'elle attend la base de données. Ces threads coûtent cher, on les limite donc avec un pool, et c'est souvent ce pool qui fixe le nombre de requêtes traitées en même temps. Les virtual threads, finalisés en Java 21, permettent de continuer à écrire du code bloquant tout simple, avec un thread par tâche, même quand il y en a des dizaines de milliers.
<!--more-->

Dans cet article :
- Un thread par requête
- Le fonctionnement des virtual threads
- Créer des virtual threads
- Un serveur HTTP sur des virtual threads
- Lancer des appels en parallèle
- Migrer du code existant
- Le pinning avec synchronized
- Les ThreadLocal
- Les tâches de calcul

Pré-requis : Java 21 ou plus récent. Les virtual threads ont été en preview dans Java 19 et 20, avant d'être finalisés en Java 21 par la JEP 444. Les mesures de l'article ont été faites avec OpenJDK 21.0.12 sur une VM Linux à 4 vCPU (Xeon 2,1 GHz) et 16 Go de RAM : les chiffres seront différents sur votre machine, ce sont les ordres de grandeur qui comptent.

## Un thread par requête

Dans le modèle *thread-per-request*, chaque requête garde son thread jusqu'à la réponse. Avec un pool de threads, cela donne :

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

Un thread Java classique, qu'on appelle maintenant *platform thread*, est une enveloppe autour d'un thread du système d'exploitation. Sous Linux x64, la JVM lui réserve une pile de 1 Mo par défaut (option `-Xss`). Cette mémoire est réservée, pas consommée : seules les pages réellement utilisées par la pile occupent de la RAM. Le coût reste bien réel. Pour s'en rendre compte, on peut démarrer un grand nombre de threads qui attendent tous sur un `CountDownLatch`, et mesurer le temps de démarrage et la mémoire résidente du processus :

|                           | 10 000 platform threads | 10 000 virtual threads | 1 000 000 virtual threads |
|---------------------------|-------------------------|------------------------|---------------------------|
| Temps de démarrage        | 4 à 6 s                 | environ 70 ms          | 1,3 à 2,6 s               |
| Mémoire résidente en plus | environ 250 Mo          | environ 26 Mo          | environ 1 Go              |

Les threads de ce test ont une pile presque vide, alors qu'un thread qui traverse toutes les couches d'un framework web en utilise beaucoup plus. Avec des platform threads, on se limite donc en général à quelques centaines ou quelques milliers de threads. Et ce n'est pas le processeur qui plafonne : pendant les entrées/sorties, il n'a rien à faire.

Avant Java 21, pour dépasser cette limite, il fallait passer à la programmation asynchrone (`CompletableFuture`, Reactor, RxJava) ou à un framework à boucle d'événements (Netty, Vert.x). C'est très efficace, mais le code ne s'écrit plus du tout de la même façon, et les piles d'appels deviennent difficiles à lire quand il faut déboguer. Avec les virtual threads, on garde le code bloquant habituel, sans le coût des platform threads.

## Le fonctionnement des virtual threads

Un virtual thread est un thread géré par la JVM et non par le système. Pour s'exécuter, il est monté sur un platform thread, qu'on appelle *carrier thread*. Quand il se bloque (entrée/sortie, `sleep`, attente d'un verrou), la JVM le démonte : sa pile est rangée dans le tas Java, et le carrier peut faire tourner un autre virtual thread. Quand l'opération bloquante se termine, le virtual thread est remonté, pas forcément sur le même carrier :

```
Carrier 1 :  [VT-1 exécute] [VT-3 exécute] [VT-1 reprend] [VT-5 exécute]
Carrier 2 :  [VT-2 exécute] [VT-4 exécute] [VT-2 reprend] [VT-6 exécute]
```

Les carriers sont fournis par un `ForkJoinPool` dédié, en mode FIFO. Par défaut, il y en a autant que de processeurs disponibles (propriété `jdk.virtualThreadScheduler.parallelism`), et le pool peut grossir temporairement jusqu'à 256 threads (`jdk.virtualThreadScheduler.maxPoolSize`) pour compenser certains blocages. Quelques carriers suffisent donc à faire tourner des centaines de milliers de virtual threads, tant que ceux-ci passent leur temps à attendre.

Attention, un virtual thread n'est pas plus rapide qu'un platform thread : il exécute le même code, à la même vitesse. Ce qu'on gagne, c'est le nombre de tâches qui peuvent attendre en même temps. Le planificateur ne fait pas non plus de partage de temps : un virtual thread qui calcule sans jamais se bloquer garde son carrier jusqu'au bout (on y revient dans la dernière section).

## Créer des virtual threads

Le plus direct est `Thread.startVirtualThread` :

```java
Thread vt = Thread.startVirtualThread(() -> {
    System.out.println("Je tourne sur un virtual thread : " + Thread.currentThread());
});

vt.join();  // Attendre la fin
```

```
Je tourne sur un virtual thread : VirtualThread[#19]/runnable@ForkJoinPool-1-worker-1
```

Le `toString()` indique le carrier sur lequel le virtual thread est monté (`ForkJoinPool-1-worker-1`). Le `join()` n'est pas là que pour l'exemple : les virtual threads sont toujours des threads démons, la JVM n'attend pas qu'ils se terminent pour s'arrêter.

`Thread.ofVirtual()` renvoie un *builder*, qui permet de donner un nom au thread (un virtual thread n'en a pas par défaut) ou de lui associer un gestionnaire d'exceptions :

```java
Thread vt = Thread.ofVirtual()
    .name("worker-", 0)  // Nommage avec compteur : worker-0, worker-1...
    .uncaughtExceptionHandler((t, e) -> System.err.println(t.getName() + " : " + e))
    .start(() -> {
        System.out.println("Virtual thread nommé : " + Thread.currentThread().getName());
    });

vt.join();
```

```
Virtual thread nommé : worker-0
```

En pratique, on passe surtout par un `ExecutorService`. `Executors.newVirtualThreadPerTaskExecutor()` crée un nouveau virtual thread pour chaque tâche soumise :

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

Le `try` avec ressource attend la fin de toutes les tâches à la fermeture de l'executor (`ExecutorService` est `AutoCloseable` depuis Java 19). Ces 10 000 tâches dorment chacune une seconde, et le programme se termine en 1,1 s environ sur la VM de test, car un virtual thread endormi ne bloque aucun carrier. Avec `Executors.newFixedThreadPool(200)`, il faut 50 s : les tâches passent 200 par 200. Avec `Executors.newCachedThreadPool()`, qui crée un nouveau platform thread dès qu'aucun n'est libre, il faut 4,8 s, passées surtout à créer puis à arrêter environ 6 000 threads.

Enfin, pour du code existant qui attend une `ThreadFactory`, `Thread.ofVirtual().factory()` en fournit une :

```java
ThreadFactory factory = Thread.ofVirtual()
    .name("vt-pool-", 0)
    .factory();

Thread t = factory.newThread(() -> System.out.println("Créé via factory"));
t.start();
```

## Un serveur HTTP sur des virtual threads

Le serveur HTTP intégré au JDK (`com.sun.net.httpserver`) accepte n'importe quel `Executor`. Voici un serveur dont chaque requête simule un appel de 100 ms à une base de données, avec un pool classique de 200 threads :

```java
void startServer() throws IOException {
    var server = HttpServer.create(new InetSocketAddress(8080), 0);
    server.setExecutor(Executors.newFixedThreadPool(200));  // Max 200 requêtes simultanées

    server.createContext("/api", exchange -> {
        // Simuler un appel à une base de données (100 ms)
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        byte[] response = "OK".getBytes();
        exchange.sendResponseHeaders(200, response.length);
        exchange.getResponseBody().write(response);
        exchange.close();
    });

    server.start();
}
```

Avec ce pool, 200 requêtes au maximum sont traitées en même temps : la 201e attend dans la file de l'executor qu'un thread se libère, alors que les 200 premières ne font que dormir. Pour passer aux virtual threads, seule la ligne `setExecutor` change :

```java
server.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
```

Chaque requête a maintenant son propre thread. La limite ne vient plus du nombre de threads, mais du reste du système : connexions acceptées, descripteurs de fichiers, et surtout ce qu'il y a derrière. Une base de données qui accepte 50 connexions n'en acceptera pas 10 000 parce que le serveur web en est capable.

## Lancer des appels en parallèle

Les virtual threads sont aussi pratiques pour lancer plusieurs appels bloquants en même temps et attendre leurs résultats. Ici, on agrège les réponses de quatre API :

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

Avec un `callApi` qui simule un appel de 200 ms, la méthode répond en 260 ms environ, au lieu de 800 ms pour quatre appels à la suite. Créer quatre threads à chaque appel ne pose aucun problème avec des virtual threads, alors qu'avec des platform threads, il faudrait partager un pool et le dimensionner avec soin.

Par contre, ce code a un défaut : si l'appel du prix échoue, `priceFuture.get()` lève une exception, mais les autres appels continuent de tourner, et la fermeture de l'executor attend qu'ils se terminent. C'est ce que règle la *structured concurrency* (`StructuredTaskScope`), qui interrompt les autres sous-tâches dès que l'une d'elles échoue. Elle n'est qu'en preview dans Java 21 (JEP 453) et l'est toujours dans Java 25, avec une API qui a beaucoup changé entre-temps. Elle est donc à réserver aux essais, avec l'option `--enable-preview`.

## Migrer du code existant

### Remplacer un pool de threads

La migration la plus simple consiste à remplacer le pool :

```java
// Avant
ExecutorService executor = Executors.newFixedThreadPool(200);

// Après
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
```

Cet executor ne réutilise pas ses threads : chaque tâche obtient un nouveau virtual thread, qui disparaît à la fin de la tâche. C'est voulu, un virtual thread coûte trop peu pour qu'un pool ait un intérêt. Mettre des virtual threads dans un pool fixe n'a donc pas de sens :

```java
// À éviter : un pool de virtual threads
ExecutorService pool = Executors.newFixedThreadPool(100, Thread.ofVirtual().factory());

// Correct : un virtual thread par tâche
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
```

Si le pool servait aussi à limiter le nombre d'appels simultanés vers une ressource (une base de données, une API qui limite le débit), on garde cette limite avec un `Semaphore` :

```java
private final Semaphore connexions = new Semaphore(10);  // 10 requêtes simultanées au maximum

String interrogerBase(String requete) throws InterruptedException {
    connexions.acquire();
    try {
        return executerRequete(requete);
    } finally {
        connexions.release();
    }
}
```

Toutes les applications ne gagnent pas à migrer : ce sont les traitements qui attendent beaucoup, avec de nombreuses tâches simultanées (serveur web, appels HTTP, accès aux bases de données), qui en profitent.

### Spring Boot

Depuis Spring Boot 3.2, une propriété suffit, à condition de tourner sur Java 21 ou plus :

```properties
# application.properties
spring.threads.virtual.enabled=true
```

Tomcat traite alors chaque requête sur un virtual thread, et les executors configurés par Spring Boot pour `@Async` et `@Scheduled` utilisent eux aussi des virtual threads. La documentation de Spring Boot signale un effet de bord : les virtual threads étant des threads démons, une application qui ne tourne que grâce à des tâches `@Scheduled` peut s'arrêter d'elle-même. La propriété `spring.main.keep-alive=true` évite ça.

### Les bibliothèques

Le code bloquant habituel fonctionne sans modification sur un virtual thread : `java.net.http.HttpClient`, les sockets, `ReentrantLock`, `Semaphore`, `CompletableFuture`, JDBC... En Java 21, il faut surtout savoir si une bibliothèque se bloque à l'intérieur de blocs `synchronized` (voir la section suivante). Les pilotes JDBC les plus courants ont été adaptés : PostgreSQL a remplacé ses `synchronized` par des `ReentrantLock` dans la version 42.6.0 de son pilote, MySQL Connector/J dans la version 9.0.0.

## Le pinning avec synchronized

En Java 21, un virtual thread ne peut pas être démonté de son carrier dans deux cas : quand il exécute du code dans un bloc ou une méthode `synchronized`, et quand il exécute une méthode native (JNI) ou une fonction étrangère (FFM). On dit qu'il est épinglé (*pinned*). S'il se bloque à ce moment-là, il bloque aussi son carrier, qui ne peut plus faire tourner d'autres virtual threads.

Le programme suivant le met en évidence. Il lance deux fois plus de tâches que de processeurs, et chaque tâche dort une seconde en tenant un verrou. La mesure est faite une première fois avec `synchronized`, puis avec un `ReentrantLock`. Chaque tâche a son propre verrou, elles ne s'attendent donc jamais entre elles :

```java
import java.time.Duration;
import java.util.concurrent.Executors;
import java.util.concurrent.locks.ReentrantLock;

public class Pinning {
    public static void main(String[] args) {
        int nbTaches = 2 * Runtime.getRuntime().availableProcessors();
        mesurer("synchronized", nbTaches, () -> {
            Object verrou = new Object();  // un verrou par tâche : aucune contention
            synchronized (verrou) {
                dormir();
            }
        });
        mesurer("ReentrantLock", nbTaches, () -> {
            ReentrantLock verrou = new ReentrantLock();
            verrou.lock();
            try {
                dormir();
            } finally {
                verrou.unlock();
            }
        });
    }

    static void mesurer(String nom, int nbTaches, Runnable tache) {
        long debut = System.nanoTime();
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < nbTaches; i++) {
                executor.submit(tache);
            }
        }
        System.out.printf("%s : %d tâches en %d ms%n", nom, nbTaches,
            Duration.ofNanos(System.nanoTime() - debut).toMillis());
    }

    static void dormir() {
        try {
            Thread.sleep(Duration.ofSeconds(1));
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

Avec Java 21, sur la VM de test (4 processeurs, donc 4 carriers) :

```
synchronized : 8 tâches en 2012 ms
ReentrantLock : 8 tâches en 1001 ms
```

Avec `synchronized`, les 4 premières tâches bloquent les 4 carriers pendant leur `sleep`, et les 4 suivantes attendent leur tour. Avec `ReentrantLock`, le virtual thread est démonté pendant le `sleep` et les 8 tâches dorment en même temps.

En Java 21, la parade consiste donc à remplacer `synchronized` par un `ReentrantLock` quand le bloc contient une opération bloquante (entrée/sortie, appel réseau) et qu'il est exécuté souvent. Un `synchronized` qui protège une simple opération en mémoire ne pose pas de problème : l'épinglage ne coûte quelque chose que si le thread se bloque pendant ce temps.

Pour repérer ces blocages, Java 21 propose l'option `-Djdk.tracePinnedThreads=short` (ou `full` pour avoir la pile d'appels complète), qui affiche un message quand un virtual thread se bloque alors qu'il est épinglé :

```
VirtualThread[#19]/runnable@ForkJoinPool-1-worker-1 reason:MONITOR
    Pinning.lambda$main$0(Pinning.java:11) <== monitors:1
```

L'événement JFR `jdk.VirtualThreadPinned` donne la même information. Il est activé par défaut dans un enregistrement Java Flight Recorder, pour les blocages de plus de 20 ms :

```bash
java -XX:StartFlightRecording=filename=app.jfr -jar app.jar
jfr print --events jdk.VirtualThreadPinned app.jfr
```

Depuis Java 24 (JEP 491), `synchronized` n'épingle plus le virtual thread dans la plupart des cas : il peut être démonté même s'il détient un moniteur. Le même programme lancé avec Java 25 donne :

```
synchronized : 8 tâches en 1020 ms
ReentrantLock : 8 tâches en 1002 ms
```

La propriété `jdk.tracePinnedThreads` a été supprimée avec ce changement : Java 24 et les versions suivantes l'ignorent, sans message. L'événement JFR, lui, est toujours là. L'épinglage n'a d'ailleurs pas complètement disparu : un virtual thread qui exécute du code natif garde son carrier, et un appel natif bloquant (`usleep` appelé via l'API FFM, par exemple) bloque toujours le carrier avec Java 25. Sur Java 21 (LTS), qui n'inclut pas la JEP 491, les conseils de cette section restent valables.

## Les ThreadLocal

Les `ThreadLocal` fonctionnent avec les virtual threads, mais un usage courant devient contre-productif : garder par thread un objet coûteux à créer.

```java
// À éviter : un buffer de 1 Mo par thread
private static final ThreadLocal<byte[]> BUFFER = ThreadLocal.withInitial(() -> new byte[1024 * 1024]);
```

Avec un pool de 200 threads, ce code crée au plus 200 buffers, réutilisés d'une tâche à l'autre. Avec un virtual thread par tâche, chaque tâche crée son propre buffer, qui ne sert qu'une fois : 10 000 requêtes simultanées, et ce sont 10 Go de buffers. Mieux vaut alors allouer l'objet quand on en a besoin, ou partager une instance thread-safe. Pour repérer ces usages, l'option `-Djdk.traceVirtualThreadLocals=true` affiche une pile d'appels à chaque fois qu'un virtual thread donne une valeur à un `ThreadLocal`.

Pour transmettre un contexte (utilisateur connecté, identifiant de requête) d'un appel à l'autre sans le passer en paramètre, le JDK propose une alternative aux `ThreadLocal`, les `ScopedValue`. La valeur est liée le temps de l'exécution d'une méthode, ne peut pas être modifiée, et redevient non liée à la fin :

```java
private static final ScopedValue<RequestContext> CONTEXT = ScopedValue.newInstance();

ScopedValue.where(CONTEXT, new RequestContext(userId))
    .run(() -> {
        // CONTEXT.get() retourne le RequestContext
        processRequest();
    });
```

Attention, les `ScopedValue` ne sont qu'en preview dans Java 21 (JEP 446) : il faut compiler et lancer le programme avec `--enable-preview`. Elles sont finalisées dans Java 25, où ce code fonctionne sans option.

## Les tâches de calcul

Les virtual threads sont faits pour les tâches qui passent leur temps à attendre. La Javadoc de `Thread` précise d'ailleurs qu'ils ne sont pas prévus pour les calculs longs. Une tâche de calcul ne se bloque jamais : son virtual thread occupe son carrier du début à la fin, et n'apporte rien par rapport à un platform thread. Comme le planificateur ne fait pas de partage de temps, une telle tâche peut même retarder les autres. Avec un seul carrier (`-Djdk.virtualThreadScheduler.parallelism=1`), un calcul de 2 secondes empêche toute autre tâche de démarrer pendant ces 2 secondes.

Pour ce type de travail, on garde un pool de platform threads dimensionné sur le nombre de processeurs :

```java
// À éviter : calcul CPU-bound, le virtual thread ne se bloque jamais
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> computeFibonacci(1_000_000));  // Monopolise un carrier
}

// Correct : un pool de platform threads dimensionné au nombre de processeurs
ExecutorService cpuExecutor = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());
cpuExecutor.submit(() -> computeFibonacci(1_000_000));
```

Les platform threads ne disparaissent donc pas : ils restent le bon choix pour le calcul, et pour les quelques threads de fond qui vivent aussi longtemps que l'application.

## Voir aussi

- [Spring : Comment utiliser les application properties]({% post_url 2021-04-29-Comment-utiliser-les-properties-spring %})
- [Les files (Queue) et Deques en Java]({% post_url 2025-09-26-Framework-collections-java-queue %})
- [JEP 444 : Virtual Threads](https://openjdk.org/jeps/444)
- [JEP 491 : Synchronize Virtual Threads without Pinning](https://openjdk.org/jeps/491)
- [JEP 453 : Structured Concurrency (Preview)](https://openjdk.org/jeps/453)
- [JEP 446 : Scoped Values (Preview)](https://openjdk.org/jeps/446)
- [Virtual Threads, le guide Oracle pour Java 21](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html)
