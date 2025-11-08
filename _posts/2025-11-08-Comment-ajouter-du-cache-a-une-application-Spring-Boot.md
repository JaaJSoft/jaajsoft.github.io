---
layout: article
title: "Comment ajouter du cache à une application Spring Boot"
author: Pierre Chopinet
tags:
  - java
  - spring
  - spring-boot
  - cache
  - performance
  - redis
  - caffeine
---

Le cache est l’un des leviers les plus efficaces pour améliorer la latence et réduire la charge d’une application. Spring Boot fournit une abstraction de cache très puissante, compatible avec plusieurs moteurs (Caffeine, Redis, Ehcache, Hazelcast, etc.).

Dans cet article, vous verrez comment activer le cache, choisir un moteur, utiliser les annotations `@Cacheable`, `@CacheEvict`, `@CachePut`, définir les clés et TTL, exposer des métriques et mettre en place une stratégie d’invalidation.
<!--more-->

---

## 1) Pourquoi mettre du cache ?

- Diminuer la latence des endpoints et batchs.
- Réduire la charge CPU/IO de services internes ou bases de données.
- Lisser les pics de trafic et améliorer la résilience.
- Faire des économies d’infrastructure.

Attention : le cache n’est pas un substitut à un modèle de données ou d’indexation correct. Il complète une conception saine.

---

## 2) Panorama rapide de l’abstraction Spring Cache

L’API Spring Cache fournit :

- Des annotations déclaratives : `@EnableCaching`, `@Cacheable`, `@CacheEvict`, `@CachePut`, `@Caching`.
- Un mécanisme de génération de clé (SpEL) et de conditions (`condition`, `unless`).
- Une intégration transparente avec différents `CacheManager` (Caffeine, Redis, Ehcache…).

Le code métier reste identique ; seul le backend change via la configuration.

---

## 3) Démarrage rapide

### 3.1) Dépendances Maven

```xml
<dependencies>
  <!-- API cache Spring -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
    <version>3.3.5</version>
  </dependency>

  <!-- Choisissez un moteur -->
  <!-- Caffeine (en mémoire, très rapide, TTL/size policy) -->
  <dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
    <version>3.1.8</version>
  </dependency>

  <!-- Ou Redis (partagé, scalable) -->
  <!--
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
    <version>3.3.5</version>
  </dependency>
  -->
</dependencies>
```

Gradle (Kotlin DSL) :

```kotlin
dependencies {
  implementation("org.springframework.boot:spring-boot-starter-cache:3.3.5")
  implementation("com.github.ben-manes.caffeine:caffeine:3.1.8")
  // implementation("org.springframework.boot:spring-boot-starter-data-redis:3.3.5")
}
```

### 3.2) Activer le cache

Dans votre classe d’application (ou une classe de config) :

```java
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@EnableCaching
@SpringBootApplication
public class Application { }
```

### 3.3) Première méthode cachée

```java
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

@Service
public class PriceService {

  // Cache "prices" par défaut ; clé générée à partir des arguments (SpEL)
  @Cacheable(cacheNames = "prices", key = "#productId")
  public Price getPrice(String productId) {
    return fetchPriceFromSlowApi(productId); // appel coûteux
  }
}
```

- Premier appel → MISS, exécution réelle et mise en cache.
- Appels suivants avec la même clé → HIT.

---

## 4) Choisir un moteur de cache

- Caffeine : en mémoire, ultra-rapide, TTL/size/expire-after-write/access, très simple en mono-process.
- Redis : partagé (cluster/containers), persistant en mémoire, TTL par entrée, idéal multi-réplicas.
- Ehcache/Hazelcast/Infinispan : alternatives JVM, parfois distribuées, selon vos contraintes.

Commencez simple : Caffeine local en dev/POC ; passez à Redis en prod multi-instances.

---

## 5) Configuration Caffeine (recommandé en local/simple prod)

`application.yml` :

```yaml
spring:
  cache:
    cache-names: [prices, products]
    caffeine:
      spec: maximumSize=10000,expireAfterWrite=10m,recordStats
```

Déclarer un `CacheManager` explicite (optionnel si vous utilisez la propriété ci-dessus) :

```java
import com.github.benmanes.caffeine.cache.Caffeine;
import org.springframework.cache.CacheManager;
import org.springframework.cache.caffeine.CaffeineCacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.concurrent.TimeUnit;

@Configuration
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

Récupérer des stats (Micrometer) :

```yaml
management:
  endpoints.web.exposure.include: ["metrics", "health"]
```
Puis consultez `/actuator/metrics/cache.gets` etc.

---

## 6) Configuration Redis (prod multi‑instances)

Dépendances : `spring-boot-starter-data-redis` (Lettuce par défaut).

`application.yml` :

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
  cache:
    type: redis
    cache-names: [prices, products]
```

Configurer TTL par cache et sérialisation :

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.cache.*;

import java.time.Duration;
import java.util.Map;

@Configuration
public class RedisCacheConfig {

  @Bean
  public RedisCacheManager redisCacheManager(RedisConnectionFactory cf) {
    GenericJackson2JsonRedisSerializer json = new GenericJackson2JsonRedisSerializer();

    RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
        .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(json))
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

Remarques :
- Les clés sont des `String` ; les valeurs sérialisées en JSON (lisibles, évolutives).
- Adaptez le TTL à la volatilité métier de la donnée.

---

## 7) Annotations essentielles et SpEL

- `@Cacheable(cacheNames, key, unless, condition)` – lit/écrit si absence.
- `@CachePut` – force l’écriture sans court-circuiter l’exécution.
- `@CacheEvict(cacheNames, key, allEntries)` – supprime ; utile après une écriture.
- `@Caching` – combiner plusieurs annotations.

Exemples :

```java
// Clé composite avec SpEL
@Cacheable(cacheNames = "productByShop", key = "#shopId + ':' + #productId")
public Product getProduct(String shopId, String productId) { ... }

// Conditionner le cache
@Cacheable(cacheNames = "prices", key = "#id", condition = "#id != null", unless = "#result == null")
public Price price(String id) { ... }

// Invalidation ciblée après update
@CacheEvict(cacheNames = "prices", key = "#p.id")
public Price updatePrice(Price p) { return repo.save(p); }

// Invalidation massive (ex: job de purge)
@CacheEvict(cacheNames = {"prices", "products"}, allEntries = true)
public void clearAllCaches() {}
```

Astuce: définissez des clés stables et explicites; évitez celles sensibles aux variations (locales, ordre de paramètres, etc.).

---

## 8) Stratégie d’invalidation

La cohérence est clé. Quelques approches :

- Invalidation au plus près des mutations : utilisez `@CacheEvict` dans les services qui écrivent.
- Écouter des événements domain (DDDD) : `@TransactionalEventListener` pour évincer après commit.
- TTL raisonnable pour limiter la dérive en cas d’oubli d’invalidation.
- Préremplissage (warmup) des caches les plus chauds au démarrage ou via un job.

Exemple avec événement :

```java
public record PriceChangedEvent(String productId) {}

@Service
public class PriceWriter {
  private final ApplicationEventPublisher publisher;
  public PriceWriter(ApplicationEventPublisher publisher) { this.publisher = publisher; }

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

---

## 9) Tests du cache

- Test unitaire du `key` SpEL et du comportement : utilisez un `CacheManager` réel (Caffeine en mémoire) via `@SpringBootTest` ou `@DataJpaTest` + import de config.
- Vidangez le cache entre scénarios si nécessaire.

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

Pour Redis en test : utilisez Testcontainers Redis ou un Redis éphémère.

---

## 10) Monitoring et métriques

Avec Actuator + Micrometer :

- `cache.gets`, `cache.puts`, `cache.evictions`, `cache.size`, latences…
- Export Prometheus/Grafana pour visualiser le taux de HIT (vise ≥ 80 % sur les chemins chauds).

```xml
<!-- pom.xml -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: ["health", "metrics", "prometheus"]
```

---

## 11) Pièges courants et bonnes pratiques

- Ne cachez pas des données hautement sensibles dans un cache partagé sans chiffrement.
- Attention à la cardinalité des clés (ex. : `vary` non contrôlé) → explosion mémoire.
- Évitez de mettre en cache des erreurs/`null` sans TTL réduit ou garde-fou (`unless`).
- Pour les applications multi-instances, évitez le cache en mémoire seul ; préférez Redis.
- Pensez au versioning de schéma de vos objets mis en cache (compatibilité JSON lors des déploiements progressifs).
- Définissez une politique de TTL par type de donnée, documentée et mesurable.

---

## Conclusion

En résumé, la mise en place du cache avec Spring Boot apporte des gains concrets
et rapides :

- Commencez simple avec Caffeine en local et mesurez les gains sur 2-3 méthodes
  coûteuses
- En production multi-instances, migrez vers Redis pour un cache partagé et
  consistant
- Mettez en place une stratégie d'invalidation au plus près des écritures avec
  `@CacheEvict`
- Définissez des TTL adaptés à la volatilité de chaque type de donnée
- Surveillez les taux de hit/miss et la taille des caches via les métriques
  Actuator
- Restez vigilant sur la cardinalité des clés et la sécurité des données
  sensibles

En suivant ces bonnes pratiques, vous améliorerez significativement les
performances tout en gardant une dette technique maîtrisée.

---

## Pour aller plus loin

- [Documentation Spring Cache](https://docs.spring.io/spring-framework/reference/integration/cache.html)
- [Spring Boot – Cache auto-configuration](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#io.caching)
- [Caffeine](https://github.com/ben-manes/caffeine)
- [Spring Data Redis](https://docs.spring.io/spring-data/redis/docs/current/reference/html/)
- [Micrometer](https://micrometer.io/)
