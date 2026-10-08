---
layout: article
title: "Comment ajouter du cache à une application Spring Boot"
description: "Ajouter du cache à une application Spring Boot avec l'abstraction Spring Cache : Caffeine, Redis, annotations, SpEL, invalidation, tests et métriques."
tags:
  - java
  - spring
  - spring-boot
  - cache
  - performance
  - redis
  - caffeine
author: Pierre Chopinet
---

Quand une méthode coûte cher (un appel à une API lente, une grosse requête SQL) et qu'elle renvoie souvent le même résultat pour les mêmes paramètres, la mettre en cache évite de refaire le travail à chaque appel. Spring propose pour ça une abstraction de cache à base d'annotations, que Spring Boot configure tout seul avec Caffeine, Redis et d'autres moteurs. Nous allons voir comment la mettre en place, la configurer, et invalider les données au bon moment.
<!--more-->

Dans cet article :
- Les dépendances
- Activer le cache
- Mettre en cache le résultat d'une méthode
- Les annotations et les clés
- Les appels internes ne passent pas par le cache
- Configurer Caffeine
- Configurer Redis
- Invalider le cache après une écriture
- Tester le cache
- Suivre le cache avec Actuator

Pré-requis : Java 17 ou plus récent et Spring Boot 3 ou 4. Les exemples ont été testés avec Spring Boot 3.5.16 et Spring Boot 4.1.1 (Java 21), les différences de la version 4 sont signalées au fil de l'article.

## Les dépendances

L'abstraction de cache fait partie de Spring Framework, et le starter `spring-boot-starter-cache` l'ajoute avec son auto-configuration. Il faut ensuite choisir où stocker les données. Caffeine garde le cache en mémoire, dans la JVM : c'est le plus rapide, mais chaque instance de l'application a son propre cache. Redis stocke le cache dans un serveur à part, partagé par toutes les instances. Spring Boot sait aussi configurer Ehcache (via JCache), Hazelcast, Infinispan, Couchbase et Cache2k.

Avec Maven :

```xml
<!-- Les versions des starters Spring Boot sont gérées par le parent BOM
     (spring-boot-starter-parent) : inutile de les préciser ici. -->
<dependencies>
  <!-- API cache Spring -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
  </dependency>

  <!-- Choisissez un moteur -->
  <!-- Caffeine (en mémoire, local à chaque instance) -->
  <dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
  </dependency>

  <!-- Ou Redis (partagé entre les instances) -->
  <!--
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
  </dependency>
  -->
</dependencies>
```

Avec Gradle (Kotlin DSL) :

```kotlin
// Avec le plugin Spring Boot / io.spring.dependency-management, les versions
// sont gérées par le BOM : on ne les précise pas.
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-cache")
  implementation("com.github.ben-manes.caffeine:caffeine")
  // implementation("org.springframework.boot:spring-boot-starter-data-redis")
}
```

Sans aucune bibliothèque de cache, Spring Boot se rabat sur une simple `ConcurrentHashMap` en mémoire. C'est pratique pour démarrer, mais la documentation de Spring Boot ne la recommande pas en production.

## Activer le cache

Le cache s'active avec l'annotation `@EnableCaching`, sur une classe de configuration :

```java
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableCaching
public class CacheConfig {
}
```

On voit souvent cette annotation directement sur la classe principale de l'application. Ça fonctionne, mais la documentation de Spring Boot le déconseille : le cache devient alors obligatoire partout, y compris dans les tests qui ne chargent qu'une partie de l'application. Dans une classe dédiée, il reste facile à isoler.

## Mettre en cache le résultat d'une méthode

Il suffit ensuite d'annoter la méthode avec `@Cacheable`. Dans les exemples, `Price` est un simple record (`public record Price(String id, BigDecimal amount) {}`) :

```java
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

@Service
public class PriceService {

  // Cache "prices", avec productId comme clé
  @Cacheable(cacheNames = "prices", key = "#productId")
  public Price getPrice(String productId) {
    return fetchPriceFromSlowApi(productId); // appel coûteux
  }
}
```

Au premier appel avec un `productId` donné, Spring exécute la méthode et range le résultat dans le cache `prices`. Aux appels suivants avec le même `productId`, il renvoie directement la valeur du cache, sans exécuter la méthode.

L'attribut `key` est une expression SpEL. Sans lui, Spring construit la clé à partir des paramètres : le paramètre lui-même s'il n'y en a qu'un, une clé composée (`SimpleKey`) s'il y en a plusieurs. Les paramètres doivent donc avoir des méthodes `equals()` et `hashCode()` correctes, ce qui est le cas des `String`, des nombres et des records.

## Les annotations et les clés

Quatre annotations couvrent la plupart des besoins :

- `@Cacheable` : renvoie la valeur du cache si elle existe, sinon exécute la méthode et met le résultat en cache
- `@CachePut` : exécute toujours la méthode, et met le cache à jour avec son résultat
- `@CacheEvict` : supprime une entrée du cache, ou tout son contenu avec `allEntries = true`
- `@Caching` : combine plusieurs de ces annotations sur une même méthode

Quelques exemples :

```java
// Clé composite avec SpEL
@Cacheable(cacheNames = "products", key = "#shopId + ':' + #productId")
public Product getProduct(String shopId, String productId) { ... }

// Ne rien mettre en cache si l'id est null, ni si le résultat est null
@Cacheable(cacheNames = "prices", key = "#id", condition = "#id != null", unless = "#result == null")
public Price price(String id) { ... }

// Mettre le cache à jour avec la valeur enregistrée
@CachePut(cacheNames = "prices", key = "#p.id")
public Price savePrice(Price p) { return repo.save(p); }

// Invalidation ciblée après update
@CacheEvict(cacheNames = "prices", key = "#p.id")
public Price updatePrice(Price p) { return repo.save(p); }

// Invalidation massive (ex: job de purge)
@CacheEvict(cacheNames = {"prices", "products"}, allEntries = true)
public void clearAllCaches() {}
```

`condition` est évaluée avant l'appel : si elle est fausse, le cache est complètement ignoré. `unless` est évaluée après, sur le résultat (`#result`), et empêche seulement la mise en cache. Sans ce `unless`, un résultat `null` est mis en cache comme les autres avec Caffeine, et les appels suivants renvoient ce `null` sans réessayer. Au passage, `#p.id` fonctionne aussi sur un record.

Pour que SpEL connaisse les noms des paramètres (`#productId`, `#p`...), le code doit être compilé avec l'option `-parameters` de `javac`. Le parent Maven de Spring Boot et son plugin Gradle l'activent. Sans elle, il faut désigner les paramètres par leur position : `#p0` ou `#a0` pour le premier.

Choisissez aussi des clés stables. Une clé qui contient une valeur différente à chaque appel (un horodatage, un identifiant de requête) crée une nouvelle entrée à chaque fois : le cache se remplit sans jamais servir.

## Les appels internes ne passent pas par le cache

Les annotations de cache fonctionnent grâce à un proxy que Spring place autour du bean : c'est lui qui intercepte l'appel et consulte le cache. Quand une méthode appelle une autre méthode du même objet, l'appel ne passe pas par ce proxy, et le cache est ignoré :

```java
public BigDecimal getPriceWithTax(String productId) {
  // Appel interne : il ne passe pas par le proxy, donc pas par le cache
  return getPrice(productId).amount().multiply(new BigDecimal("1.20"));
}
```

Si cette méthode est dans `PriceService`, chaque appel à `getPriceWithTax` rappelle l'API lente, alors que `getPrice` est annotée avec `@Cacheable`. Le plus simple est de placer la méthode mise en cache dans un autre bean, injecté là où on en a besoin. Pour la même raison, le cache n'est pas encore actif dans une méthode `@PostConstruct`.

## Configurer Caffeine

Quand Caffeine est dans le classpath, Spring Boot crée un `CaffeineCacheManager`, qui se règle dans `application.yml` :

```yaml
spring:
  cache:
    cache-names: [prices, products]
    caffeine:
      spec: maximumSize=10000,expireAfterWrite=10m,recordStats
```

La spécification limite chaque cache à 10 000 entrées, fait expirer une entrée 10 minutes après son écriture, et active les statistiques (`recordStats`), dont les métriques ont besoin. La taille maximale évite que le cache grossisse sans limite.

Attention, `cache-names` crée les caches au démarrage, mais fige aussi la liste : une méthode qui utilise un cache absent de la liste échoue au premier appel avec `IllegalArgumentException: Cannot find cache named '...'`. Sans `cache-names`, Caffeine crée les caches à la demande.

Si Caffeine et Redis sont tous les deux présents dans le classpath, Spring Boot choisit Redis, car il teste les moteurs dans un ordre fixe. On force alors le choix avec `spring.cache.type: caffeine`.

On peut aussi déclarer soi-même le `CacheManager`, par exemple dans la classe `CacheConfig` vue plus haut :

```java
import com.github.benmanes.caffeine.cache.Caffeine;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.cache.caffeine.CaffeineCacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.concurrent.TimeUnit;

@Configuration
@EnableCaching
public class CacheConfig {

  @Bean
  public CacheManager cacheManager() {
    CaffeineCacheManager mgr = new CaffeineCacheManager("prices", "products");
    mgr.setCaffeine(Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(10, TimeUnit.MINUTES)
        .recordStats());
    return mgr;
  }
}
```

C'est l'un ou l'autre : dès qu'un bean `CacheManager` existe, Spring Boot n'en crée plus, et les propriétés `spring.cache.*` ne sont plus utilisées.

Caffeine a une limite : le cache reste local à chaque instance. Si l'application tourne sur trois instances, une entrée invalidée sur l'une reste en cache sur les deux autres jusqu'à son expiration. Dans ce cas, il faut un cache partagé.

## Configurer Redis

Avec `spring-boot-starter-data-redis`, Spring Boot crée un `RedisCacheManager`. La configuration minimale indique où trouver Redis et la durée de vie des entrées :

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
  cache:
    type: redis
    cache-names: [prices, products]
    redis:
      time-to-live: 10m
```

Par défaut, les valeurs sont stockées avec la sérialisation Java : les objets mis en cache doivent implémenter `Serializable`, sinon la mise en cache lève une exception. Pour stocker du JSON et régler une durée de vie différente par cache, on déclare son propre `RedisCacheManager` :

```java
import java.time.Duration;
import java.util.Map;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.serializer.RedisSerializationContext.SerializationPair;
import org.springframework.data.redis.serializer.RedisSerializer;

@Configuration
public class RedisCacheConfig {

  @Bean
  public RedisCacheManager redisCacheManager(RedisConnectionFactory cf) {
    RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
        .serializeValuesWith(SerializationPair.fromSerializer(RedisSerializer.json()))
        .entryTtl(Duration.ofMinutes(10)); // TTL par défaut

    Map<String, RedisCacheConfiguration> configs = Map.of(
        "prices", defaultConfig.entryTtl(Duration.ofMinutes(15)),
        "products", defaultConfig.entryTtl(Duration.ofMinutes(5))
    );

    return RedisCacheManager.builder(cf)
        .cacheDefaults(defaultConfig)
        .withInitialCacheConfigurations(configs)
        .build();
  }
}
```

`RedisSerializer.json()` renvoie le sérialiseur JSON de Spring Data Redis, basé sur Jackson. Jackson n'est pas inclus dans le starter Redis : il est déjà présent dans une application web (`spring-boot-starter-web`), sinon on ajoute `spring-boot-starter-json`. Beaucoup d'exemples utilisent directement `new GenericJackson2JsonRedisSerializer()` : ça fonctionne avec Spring Boot 3, mais cette classe est dépréciée dans Spring Data Redis 4 (Spring Boot 4), qui passe à Jackson 3. Comme Spring Boot 4 n'embarque plus Jackson 2 par défaut, l'application ne démarre même pas (`NoClassDefFoundError: com/fasterxml/jackson/databind/...`). `RedisSerializer.json()` choisit la bonne implémentation dans les deux versions.

Les clés sont préfixées par le nom du cache suivi de `::`. Après un appel à `getPrice("A-42")`, on retrouve l'entrée dans Redis :

```
$ redis-cli GET "prices::A-42"
"{\"@class\":\"com.example.cache.Price\",\"id\":\"A-42\",\"amount\":[\"java.math.BigDecimal\",9.99]}"
$ redis-cli TTL "prices::A-42"
(integer) 889
```

La durée de vie de 15 minutes du cache `prices` s'applique bien (900 secondes au moment de l'écriture). Le JSON contient le nom de la classe (`@class`), dont Jackson se sert pour recréer le bon objet à la lecture. La documentation de Spring Data Redis prévient que ce mécanisme est dangereux avec des données venant d'une source non fiable : ce Redis doit rester réservé à vos applications. Pensez aussi qu'une entrée écrite par une ancienne version de l'application peut ne plus se relire si la classe a changé entre deux déploiements.

Comme pour Caffeine, ce bean remplace l'auto-configuration : `spring.cache.cache-names` et `spring.cache.redis.*` ne sont plus pris en compte. Ici, le cache `products` garde 5 minutes, et tout autre cache prend les 10 minutes de `defaultConfig`. Pour seulement ajuster le `RedisCacheManager` créé par Spring Boot, il existe aussi un bean `RedisCacheManagerBuilderCustomizer`.

Avec Spring Boot 4, les écritures dans le cache Redis sont en plus asynchrones par défaut : une valeur mise en cache ou supprimée peut n'être visible dans Redis qu'un court instant après l'appel. Pour un cache qui a besoin d'écritures immédiates, on construit le gestionnaire avec `RedisCacheManager.builder(RedisCacheWriter.create(cf, writer -> writer.immediateWrites()))`.

## Invalider le cache après une écriture

Un cache n'est utile que s'il ne renvoie pas de données périmées. La règle de base est d'invalider au plus près des écritures : chaque méthode qui modifie une donnée porte un `@CacheEvict` (ou un `@CachePut`) sur l'entrée concernée. La durée de vie sert de filet de sécurité si une invalidation a été oubliée, et se règle selon la fréquence à laquelle la donnée change.

Quand l'écriture se fait dans une transaction, mieux vaut n'invalider qu'une fois la transaction validée : si l'entrée est supprimée avant le commit, une autre requête peut relire l'ancienne valeur en base et la remettre en cache. Une solution est de publier un événement et de l'écouter avec `@TransactionalEventListener` :

```java
public record PriceChangedEvent(String productId) {}

@Service
public class PriceWriter {
  private final PriceRepository repo;
  private final ApplicationEventPublisher publisher;

  public PriceWriter(PriceRepository repo, ApplicationEventPublisher publisher) {
    this.repo = repo;
    this.publisher = publisher;
  }

  @Transactional
  public void updatePrice(Price p) {
    repo.save(p);
    publisher.publishEvent(new PriceChangedEvent(p.id()));
  }
}

@Component
public class PriceCacheInvalidator {
  @CacheEvict(cacheNames = "prices", key = "#event.productId")
  @TransactionalEventListener
  public void onPriceChanged(PriceChangedEvent event) {}
}
```

Par défaut, l'écouteur est appelé après le commit (phase `AFTER_COMMIT`) : en cas de rollback, le cache n'est pas touché. Attention, si l'événement est publié en dehors de toute transaction, l'écouteur n'est pas appelé du tout, et l'entrée reste en cache. Pour l'appeler quand même dans ce cas, on ajoute `fallbackExecution = true` à l'annotation.

## Tester le cache

Pour vérifier qu'une méthode est bien mise en cache, le plus simple est un test `@SpringBootTest`, qui utilise le vrai `CacheManager` :

```java
@SpringBootTest
class PriceServiceTest {

  @Autowired PriceService service;
  @Autowired CacheManager cacheManager;

  @Test
  void cached_method_hits_cache() {
    String id = "A-42";
    service.getPrice(id); // MISS
    service.getPrice(id); // HIT

    var cache = cacheManager.getCache("prices");
    assertThat(cache).isNotNull();
    assertThat(cache.get(id)).isNotNull();
  }
}
```

Ce test vérifie que le résultat est rangé sous la clé attendue. Pour vérifier en plus que l'appel coûteux n'a lieu qu'une fois, on remplace le bean qui fait cet appel par un mock avec `@MockitoBean`, et on compte ses appels avec `verify(api, times(1))`.

Attention, Spring réutilise le même contexte, et donc les mêmes caches, entre les tests qui ont la même configuration. Pour que chaque test parte d'un cache vide :

```java
@BeforeEach
void clearCaches() {
  cacheManager.getCacheNames().forEach(name -> cacheManager.getCache(name).clear());
}
```

Pour tester avec Redis, Testcontainers permet de lancer un vrai serveur Redis dans un conteneur le temps des tests.

## Suivre le cache avec Actuator

Pour savoir si le cache sert vraiment, Spring Boot Actuator publie des métriques pour chaque cache. On ajoute la dépendance :

```xml
<!-- pom.xml -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Puis on expose l'endpoint `metrics` :

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "health,metrics"
```

Après trois appels à `getPrice("A-42")`, le premier qui rate le cache et deux qui le trouvent, on interroge la métrique `cache.gets` en filtrant sur le cache `prices` et les succès (`hit`) :

```bash
curl -s "http://localhost:8080/actuator/metrics/cache.gets?tag=cache:prices&tag=result:hit" | jq '.measurements'
```

```json
[
  {
    "statistic": "COUNT",
    "value": 2.0
  }
]
```

Avec `result:miss`, on obtient 1. Le rapport entre les deux donne le taux de succès du cache. On trouve aussi `cache.size` et `cache.evictions`.

Ces compteurs ont besoin des statistiques du moteur. Sans `recordStats` dans la spécification Caffeine, seule `cache.size` est publiée, et Micrometer le signale au démarrage par un avertissement `The cache 'prices' is not recording statistics`. Avec Redis, il faut ajouter `spring.cache.redis.enable-statistics: true`, ou appeler `enableStatistics()` sur le builder si on déclare son propre `RedisCacheManager`. Enfin, seuls les caches qui existent au démarrage sont suivis : un cache créé à la volée au premier appel n'apparaît pas dans les métriques. C'est une raison de plus de lister ses caches dans `cache-names`.

Pour envoyer ces métriques vers Prometheus et les afficher dans Grafana, il suffit d'ajouter la dépendance `micrometer-registry-prometheus` et d'exposer aussi l'endpoint `prometheus`.

Voilà, vous savez maintenant ajouter du cache à une application Spring Boot. Commencez par une ou deux méthodes vraiment coûteuses, et vérifiez avec ces métriques que le cache est bien utilisé avant d'aller plus loin.

## Voir aussi

- [Comment ajouter du cache à une application Django]({% post_url 2025-11-01-Comment-ajouter-du-cache-a-une-application-Django %})
- [Comment ajouter un cache à une application Flask]({% post_url 2025-09-14-Comment-utiliser-un-cache-avec-Flask %})
- [Ajouter un cache à notre application FastAPI avec redis]({% post_url 2025-08-18-Utiliser-fastapi-cache2-avec-FastAPI %})
- [Spring : Comment utiliser les application properties]({% post_url 2021-04-29-Comment-utiliser-les-properties-spring %})
- [Documentation Spring Framework sur le cache](https://docs.spring.io/spring-framework/reference/integration/cache.html)
- [Documentation Spring Boot sur le cache](https://docs.spring.io/spring-boot/reference/io/caching.html)
- [Spring Data Redis : le cache Redis](https://docs.spring.io/spring-data/redis/reference/redis/redis-cache.html)
- [Caffeine](https://github.com/ben-manes/caffeine)
