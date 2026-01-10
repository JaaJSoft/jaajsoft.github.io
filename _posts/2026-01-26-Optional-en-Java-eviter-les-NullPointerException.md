---
layout: article
title: "Optional en Java : éviter les NullPointerException"
author: Pierre Chopinet
tags:
  - java
  - optional
  - "null"
---

`Optional<T>` est un conteneur introduit en Java 8 pour gérer explicitement l'absence de valeur et éviter les redoutées `NullPointerException`. Au lieu de retourner `null`, vous retournez un `Optional` qui peut être vide ou contenir une valeur.
<!--more-->

Dans cet article, vous découvrirez :
- Ce qu'est `Optional` et pourquoi l'utiliser
- Comment créer et manipuler des `Optional`
- Les méthodes essentielles (`map`, `flatMap`, `filter`, `orElse`, `ifPresent`)
- Les anti-patterns à éviter
- Les bonnes pratiques et cas d'usage
- L'intégration avec Spring Data, Stream API et records

---

## 1) Le problème : NullPointerException

`NullPointerException` (NPE) est l'erreur la plus fréquente en Java. Elle survient quand on tente d'accéder à une méthode ou un champ sur une référence `null`.

### Exemple classique sans Optional

```java
public String getUserEmail(Long userId) {
    User user = userRepository.findById(userId);
    if (user != null) {
        Address address = user.getAddress();
        if (address != null) {
            return address.getEmail();
        }
    }
    return "unknown@example.com";
}
```

**Problèmes** :
- Code verbeux avec des checks `!= null` imbriqués
- Facile d'oublier un check et déclencher une NPE
- Pas d'indication explicite qu'une valeur peut être absente

---

## 2) La solution : Optional<T>

`Optional<T>` est un conteneur qui :
- Contient une valeur de type `T` (Optional "présent")
- Ou ne contient rien (Optional "vide")

```java
import java.util.Optional;

public Optional<String> getUserEmail(Long userId) {
    return userRepository.findById(userId)
        .map(User::getAddress)
        .map(Address::getEmail);
}
```

**Avantages** :
- Code concis et lisible
- Type-safe : le compilateur force la gestion de l'absence
- Moins de NPE en production

---

## 3) Créer un Optional

### 3.1) Optional.of(value)

Crée un Optional contenant `value`. **Lance une NPE si `value` est `null`**.

```java
Optional<String> opt = Optional.of("Hello");
// Optional<String> opt = Optional.of(null); // NPE !
```

**Usage** : quand vous êtes certain que la valeur n'est jamais `null`.

### 3.2) Optional.ofNullable(value)

Crée un Optional contenant `value`, ou un Optional vide si `value` est `null`.

```java
String name = getName(); // peut retourner null
Optional<String> opt = Optional.ofNullable(name);
```

**Usage** : quand la valeur peut être `null` (cas le plus courant).

### 3.3) Optional.empty()

Crée un Optional vide.

```java
Optional<String> opt = Optional.empty();
System.out.println(opt.isPresent()); // false
```

**Usage** : pour signaler explicitement l'absence de valeur.

---

## 4) Vérifier la présence d'une valeur

### isPresent() et isEmpty()

```java
Optional<String> opt = Optional.of("Hello");

if (opt.isPresent()) {
    System.out.println(opt.get()); // Hello
}

// Java 11+
if (opt.isEmpty()) {
    System.out.println("Vide");
}
```

⚠️ **Anti-pattern** : utiliser `isPresent()` + `get()` revient à faire un check `!= null`. Préférez les méthodes fonctionnelles ci-dessous.

---

## 5) Extraire la valeur

### 5.1) get() ⚠️

Retourne la valeur si présente, sinon **lance `NoSuchElementException`**.

```java
Optional<String> opt = Optional.of("Hello");
String value = opt.get(); // Hello

Optional<String> empty = Optional.empty();
// String value = empty.get(); // NoSuchElementException !
```

**⚠️ À éviter** : préférez les méthodes sûres ci-dessous.

### 5.2) orElse(defaultValue)

Retourne la valeur si présente, sinon `defaultValue`.

```java
String name = Optional.ofNullable(getName())
    .orElse("Anonyme");
```

**Attention** : `defaultValue` est **toujours évaluée**, même si l'Optional est présent.

### 5.3) orElseGet(Supplier)

Retourne la valeur si présente, sinon appelle le `Supplier`.

```java
String name = Optional.ofNullable(getName())
    .orElseGet(() -> fetchDefaultName()); // appelé seulement si absent
```

**⚡ Performance** : préférez `orElseGet` si le calcul de la valeur par défaut est coûteux.

### 5.4) orElseThrow()

Lance une exception si l'Optional est vide.

```java
User user = userRepository.findById(id)
    .orElseThrow(() -> new UserNotFoundException("User " + id + " not found"));
```

**Usage** : quand l'absence de valeur est une erreur métier.

---

## 6) Transformation avec map()

`map()` applique une fonction à la valeur si présente, retourne un Optional du résultat.

```java
Optional<String> name = Optional.of("alice");
Optional<String> upper = name.map(String::toUpperCase);
// Optional["ALICE"]

Optional<Integer> length = name.map(String::length);
// Optional[5]
```

**Chaînage** :

```java
Optional<User> user = findUser(id);
Optional<String> email = user
    .map(User::getAddress)
    .map(Address::getEmail)
    .map(String::toLowerCase);
```

Si `user`, `getAddress()` ou `getEmail()` retourne `null` ou Optional vide, la chaîne retourne `Optional.empty()`.

---

## 7) Aplatissement avec flatMap()

`flatMap()` est utilisé quand la fonction retourne déjà un `Optional`.

### Problème avec map()

```java
Optional<User> user = findUser(id); // retourne Optional<User>

// map retourne Optional<Optional<Address>>
Optional<Optional<Address>> address = user.map(User::getOptionalAddress);
```

### Solution avec flatMap()

```java
Optional<Address> address = user.flatMap(User::getOptionalAddress);
// retourne directement Optional<Address>
```

**Exemple complet** :

```java
public Optional<String> getUserCityName(Long userId) {
    return userRepository.findById(userId)           // Optional<User>
        .flatMap(User::getAddress)                   // Optional<Address>
        .flatMap(Address::getCity)                   // Optional<City>
        .map(City::getName);                         // Optional<String>
}
```

---

## 8) Filtrage avec filter()

`filter()` garde la valeur si elle satisfait le prédicat, sinon retourne `Optional.empty()`.

```java
Optional<String> name = Optional.of("Alice");

Optional<String> longName = name.filter(n -> n.length() > 3);
// Optional["Alice"]

Optional<String> shortName = name.filter(n -> n.length() > 10);
// Optional.empty
```

**Cas d'usage** : validation conditionnelle.

```java
public Optional<User> getActiveUser(Long id) {
    return userRepository.findById(id)
        .filter(User::isActive);
}
```

---

## 9) Exécuter une action avec ifPresent()

`ifPresent(Consumer)` exécute le `Consumer` si la valeur est présente.

```java
Optional<User> user = findUser(id);
user.ifPresent(u -> System.out.println("User: " + u.getName()));
```

### ifPresentOrElse() (Java 9+)

```java
user.ifPresentOrElse(
    u -> System.out.println("User: " + u.getName()),
    () -> System.out.println("User not found")
);
```

---

## 10) Combinaison avec or() (Java 9+)

`or(Supplier<Optional>)` retourne l'Optional si présent, sinon appelle le `Supplier`.

```java
Optional<User> user = findUserInCache(id)
    .or(() -> findUserInDatabase(id))
    .or(() -> findUserInBackup(id));
```

Équivalent à un fallback en cascade.

---

## 11) Conversion en Stream (Java 9+)

`stream()` convertit un `Optional` en `Stream` de 0 ou 1 élément.

```java
List<String> emails = users.stream()
    .map(User::getEmail)                    // Stream<Optional<String>>
    .flatMap(Optional::stream)              // Stream<String> (filtre les empty)
    .collect(Collectors.toList());
```

Avant Java 9, on utilisait :

```java
.filter(Optional::isPresent)
.map(Optional::get)
```

---

## 12) Anti-patterns à éviter

### ❌ 1. Utiliser get() sans vérification

**Mauvais** :
```java
String name = optional.get(); // peut lancer NoSuchElementException
```

**Bon** :
```java
String name = optional.orElse("default");
String name = optional.orElseThrow(() -> new RuntimeException("Absent"));
```

### ❌ 2. isPresent() + get() (check null déguisé)

**Mauvais** :
```java
if (optional.isPresent()) {
    return optional.get();
}
return "default";
```

**Bon** :
```java
return optional.orElse("default");
```

### ❌ 3. Optional imbriqués : Optional<Optional<T>>

**Mauvais** :
```java
Optional<User> user = findUser(id);
Optional<Optional<Address>> address = user.map(User::getOptionalAddress);
```

**Bon** :
```java
Optional<Address> address = user.flatMap(User::getOptionalAddress);
```

### ❌ 4. Optional en paramètre de méthode

**Mauvais** :
```java
public void setName(Optional<String> name) {
    // ...
}
```

**Bon** :
```java
// Utilisez @Nullable ou surcharge
public void setName(String name) { /* ... */ }
public void setName() { /* sans nom */ }

// Ou avec annotation
public void setName(@Nullable String name) { /* ... */ }
```

### ❌ 5. Optional en champ de classe

**Mauvais** :
```java
public class User {
    private Optional<String> middleName;
}
```

**Bon** :
```java
public class User {
    private String middleName; // peut être null

    public Optional<String> getMiddleName() {
        return Optional.ofNullable(middleName);
    }
}
```

### ❌ 6. Retourner null au lieu d'Optional.empty()

**Mauvais** :
```java
public Optional<User> findUser(Long id) {
    if (notFound) {
        return null; // DANGER !
    }
    return Optional.of(user);
}
```

**Bon** :
```java
public Optional<User> findUser(Long id) {
    if (notFound) {
        return Optional.empty();
    }
    return Optional.of(user);
}
```

---

## 13) Bonnes pratiques

### ✅ À faire

1. **Retourner `Optional` dans les méthodes publiques** quand l'absence de valeur est possible et normale.

```java
public Optional<User> findUserByEmail(String email) {
    // ...
}
```

2. **Utiliser `orElseGet` pour calculs coûteux**

```java
.orElseGet(() -> database.queryDefault())
```

3. **Chaîner avec `map` / `flatMap` / `filter`**

```java
return user
    .flatMap(User::getAddress)
    .map(Address::getCity)
    .filter(city -> city.getPopulation() > 100_000)
    .orElse("Unknown");
```

4. **Ne pas retourner `Optional` de collection**, retournez une collection vide

```java
// NON
public Optional<List<User>> getUsers() { ... }

// OUI
public List<User> getUsers() {
    return users != null ? users : Collections.emptyList();
}
```

5. **Ne pas utiliser `Optional` pour des champs de classe**

```java
// NON
private Optional<String> middleName;

// OUI
private String middleName; // peut être null
```

---

## 14) Optional avec Spring Data

Spring Data JPA supporte `Optional` nativement dans les repositories.

```java
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findById(Long id);
    Optional<User> findByEmail(String email);
}
```

**Usage** :

```java
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;

    public UserDTO getUser(Long id) {
        return userRepository.findById(id)
            .map(this::toDTO)
            .orElseThrow(() -> new UserNotFoundException(id));
    }

    private UserDTO toDTO(User user) {
        return new UserDTO(user.getId(), user.getName(), user.getEmail());
    }
}
```

---

## 15) Optional avec Stream API

`Optional` s'intègre parfaitement avec les Streams.

```java
List<User> users = Arrays.asList(user1, user2, user3);

// Extraire les emails présents
List<String> emails = users.stream()
    .map(User::getEmail)              // Stream<Optional<String>>
    .flatMap(Optional::stream)        // Java 9+
    .collect(Collectors.toList());

// Trouver le premier utilisateur actif
Optional<User> firstActive = users.stream()
    .filter(User::isActive)
    .findFirst();
```

---

## 16) Optional avec Records (Java 16+)

Les records s'intègrent bien avec `Optional` pour les champs optionnels.

```java
public record UserDTO(
    Long id,
    String name,
    Optional<String> middleName,    // ❌ Anti-pattern
    String email
) {}

// Préférez :
public record UserDTO(
    Long id,
    String name,
    String middleName,  // peut être null
    String email
) {
    // Méthode accesseur pour Optional
    public Optional<String> middleName() {
        return Optional.ofNullable(middleName);
    }
}
```

Ou mieux encore, gardez le record simple :

```java
public record UserDTO(Long id, String name, String email) {}

// Classe service gère les Optional
public Optional<UserDTO> findUser(Long id) {
    return userRepository.findById(id)
        .map(user -> new UserDTO(user.getId(), user.getName(), user.getEmail()));
}
```

---

## 17) Performances

`Optional` ajoute un léger overhead (allocation d'objet). Pour du code critique en performance :

- **Évitez** `Optional` dans des boucles très fréquentes
- **Préférez** `Optional` pour les API publiques (lisibilité > micro-optimisation)
- Dans 99% des cas, l'overhead est négligeable

---

## 18) Exemples concrets

### Cas 1 : Configuration optionnelle

```java
public class AppConfig {
    private String host;
    private Integer port;

    public Optional<Integer> getPort() {
        return Optional.ofNullable(port);
    }

    public int getPortOrDefault() {
        return getPort().orElse(8080);
    }
}
```

### Cas 2 : Parsing sécurisé

```java
public Optional<Integer> parseInteger(String value) {
    try {
        return Optional.of(Integer.parseInt(value));
    } catch (NumberFormatException e) {
        return Optional.empty();
    }
}

// Usage
int port = parseInteger(input)
    .filter(p -> p > 0 && p < 65536)
    .orElse(8080);
```

### Cas 3 : Recherche en cascade

```java
public User getUser(Long id) {
    return cache.get(id)
        .or(() -> database.find(id))
        .or(() -> backup.find(id))
        .orElseThrow(() -> new UserNotFoundException(id));
}
```

### Cas 4 : Transformation conditionnelle

```java
public String formatName(User user) {
    return Optional.ofNullable(user.getMiddleName())
        .map(middle -> user.getFirstName() + " " + middle + " " + user.getLastName())
        .orElse(user.getFirstName() + " " + user.getLastName());
}
```

---

## Conclusion

`Optional` est un outil puissant pour rendre le code Java plus sûr et expressif en gérant explicitement l'absence de valeur. En suivant les bonnes pratiques, vous réduirez drastiquement les `NullPointerException` et améliorerez la lisibilité.

**Points clés à retenir :**

- Utilisez `Optional` dans les **retours de méthodes** quand l'absence est possible
- Privilégiez `map`, `flatMap`, `filter`, `orElse` plutôt que `isPresent()` + `get()`
- **Ne pas utiliser** `Optional` en paramètres de méthodes ou champs de classe
- Intégration native avec Spring Data et Stream API
- Performance acceptable pour 99% des cas

`Optional` est un incontournable du Java moderne depuis Java 8 !

---

## Pour aller plus loin

- [JDK 8 Optional Javadoc](https://docs.oracle.com/javase/8/docs/api/java/util/Optional.html)
- [JEP 303: Optional improvements (Java 9)](https://openjdk.org/jeps/303)
- [Oracle Tutorial: Optional](https://docs.oracle.com/javase/tutorial/java/javaOO/optional.html)
- [Baeldung: Guide to Java Optional](https://www.baeldung.com/java-optional)

## Voir aussi

- [Records en Java : simplifier vos DTOs]({% post_url 2026-01-10-Records-en-Java-simplifier-vos-DTOs %})
- [Pattern matching en Java moderne]({% post_url 2025-10-23-Pattern-matching-en-Java-moderne %})
- [Comment faire des group by en Java]({% post_url 2026-01-11-Comment-faire-des-group-by-en-Java %})
- [Sealed classes en Java]({% post_url 2026-01-14-Sealed-classes-en-Java %})
- [Stream API et les collections Java]({% post_url 2020-11-12-Framework-collections-java-intro %})
