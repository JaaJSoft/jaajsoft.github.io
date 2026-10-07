---
layout: article
title: "Optional en Java : éviter les NullPointerException"
tags:
  - java
  - optional
  - "null"
author: Pierre Chopinet
---

Une méthode qui renvoie `null` quand elle ne trouve rien oblige l'appelant à y penser, et le jour où il oublie, c'est la `NullPointerException`. `Optional<T>`, introduit en Java 8, rend cette absence visible dans le type de retour. Voyons comment créer et manipuler un `Optional`, et dans quels cas il vaut mieux s'en passer.
<!--more-->

Dans cet article :
- Des tests de null en cascade
- Le même code avec Optional
- Créer un Optional
- Récupérer la valeur
- Transformer avec map et flatMap
- Filtrer avec filter
- Agir sur la valeur avec ifPresent
- Chercher ailleurs avec or
- Optional et les streams
- Où ne pas utiliser Optional
- Optional avec Spring Data
- Et les performances ?

Pré-requis : Java 8 ou plus récent. Certaines méthodes demandent Java 9, 10 ou 11, c'est indiqué au cas par cas.

## Des tests de null en cascade

Voici un code qu'on croise souvent :

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

Chaque niveau peut renvoyer `null`, donc chaque niveau a son `if`. Il suffit d'en oublier un pour obtenir une `NullPointerException`, et rien dans la signature de `findById` ne prévient qu'elle peut renvoyer `null`.

## Le même code avec Optional

Un `Optional<T>` contient soit une valeur de type `T`, soit rien du tout. Si le repository renvoie un `Optional<User>` au lieu d'un `User` (c'est la signature de `findById` dans Spring Data, on y revient plus bas), le code devient :

```java
import java.util.Optional;

public Optional<String> getUserEmail(Long userId) {
    return userRepository.findById(userId)
        .map(User::getAddress)
        .map(Address::getEmail);
}
```

Si l'utilisateur n'existe pas, ou si `getAddress()` ou `getEmail()` renvoie `null`, on obtient un `Optional` vide au lieu d'une exception. Le type de retour annonce aussi à l'appelant que l'email peut manquer : pour récupérer la valeur, il doit décider quoi faire dans ce cas, par exemple avec `.orElse("unknown@example.com")` pour retrouver le comportement d'origine.

## Créer un Optional

`Optional.of()` crée un `Optional` qui contient la valeur passée. Il est réservé aux valeurs dont on est sûr qu'elles ne sont pas `null`, sinon il lance une `NullPointerException` :

```java
Optional<String> opt = Optional.of("Hello");
// Optional<String> opt = Optional.of(null); // NPE !
```

`Optional.ofNullable()` renvoie un `Optional` vide si la valeur est `null`. C'est la méthode à utiliser pour encapsuler le résultat d'une API qui peut renvoyer `null` :

```java
String name = getName(); // peut retourner null
Optional<String> opt = Optional.ofNullable(name);
```

Enfin, `Optional.empty()` crée un `Optional` vide, pour signaler explicitement l'absence de valeur :

```java
Optional<String> opt = Optional.empty();
System.out.println(opt.isPresent()); // false
```

## Récupérer la valeur

### isPresent et get

La façon la plus directe ressemble beaucoup à un test de `null` :

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

Sur un `Optional` vide, `get()` lance une `NoSuchElementException` avec le message `No value present`. Le couple `isPresent()` et `get()` revient donc à écrire un `if (x != null)` avec plus de caractères, et on perd l'intérêt d'`Optional`. Les méthodes qui suivent sont plus adaptées. La Javadoc de `get()` conseille d'ailleurs de lui préférer `orElseThrow()` sans argument (Java 10+), qui fait exactement la même chose avec un nom plus explicite.

### orElse et orElseGet

`orElse()` renvoie la valeur si elle est présente, sinon la valeur par défaut passée en argument :

```java
String name = Optional.ofNullable(getName())
    .orElse("Anonyme");
```

`orElseGet()` fait la même chose, mais la valeur par défaut est fournie par un `Supplier` :

```java
String name = Optional.ofNullable(getName())
    .orElseGet(() -> fetchDefaultName()); // appelé seulement si absent
```

La différence compte quand la valeur par défaut coûte quelque chose à calculer. L'argument de `orElse()` est évalué avant l'appel, donc même quand l'`Optional` contient une valeur. Avec une méthode qui affiche un message :

```java
static String fetchDefaultName() {
    System.out.println("fetchDefaultName() appelée");
    return "Anonyme";
}

Optional<String> name = Optional.of("Alice");
System.out.println(name.orElse(fetchDefaultName()));
System.out.println(name.orElseGet(() -> fetchDefaultName()));
```

Ce qui donne :

```
fetchDefaultName() appelée
Alice
Alice
```

Avec `orElse()`, la méthode est appelée pour rien. Sans conséquence pour une constante, mais pour une requête en base ou un calcul, préférez `orElseGet()`.

### orElseThrow

Quand l'absence de valeur est une erreur métier, `orElseThrow()` lance l'exception fournie par le `Supplier` :

```java
User user = userRepository.findById(id)
    .orElseThrow(() -> new UserNotFoundException("User " + id + " not found"));
```

Sans argument (Java 10+), `orElseThrow()` lance une `NoSuchElementException`.

## Transformer avec map et flatMap

`map()` applique une fonction à la valeur si elle est présente, et renvoie un `Optional` du résultat :

```java
Optional<String> name = Optional.of("alice");
Optional<String> upper = name.map(String::toUpperCase);
// Optional[ALICE]

Optional<Integer> length = name.map(String::length);
// Optional[5]
```

Les appels se chaînent :

```java
Optional<User> user = findUser(id);
Optional<String> email = user
    .map(User::getAddress)
    .map(Address::getEmail)
    .map(String::toLowerCase);
```

Si `user` est vide, ou si `getAddress()` ou `getEmail()` renvoie `null`, on obtient `Optional.empty` : quand la fonction passée à `map()` renvoie `null`, `map()` renvoie un `Optional` vide.

Ce modèle convient quand les getters renvoient directement l'objet, éventuellement `null`. Si un getter renvoie lui-même un `Optional`, `map()` produit un `Optional` dans un `Optional` :

```java
Optional<User> user = findUser(id); // retourne Optional<User>

// map retourne Optional<Optional<Address>>
Optional<Optional<Address>> address = user.map(User::getOptionalAddress);
```

C'est le rôle de `flatMap()`, qui aplatit le résultat :

```java
Optional<Address> address = user.flatMap(User::getOptionalAddress);
// retourne directement Optional<Address>
```

Avec un modèle où `getAddress()` et `getCity()` renvoient des `Optional`, la chaîne complète devient :

```java
public Optional<String> getUserCityName(Long userId) {
    return userRepository.findById(userId)           // Optional<User>
        .flatMap(User::getAddress)                   // Optional<Address>
        .flatMap(Address::getCity)                   // Optional<City>
        .map(City::getName);                         // Optional<String>
}
```

C'est donc le type de retour de la fonction qui décide : `map` quand elle renvoie la valeur (ou `null`), `flatMap` quand elle renvoie un `Optional`.

## Filtrer avec filter

`filter()` garde la valeur si elle respecte le prédicat, et renvoie un `Optional` vide sinon :

```java
Optional<String> name = Optional.of("Alice");

Optional<String> longName = name.filter(n -> n.length() > 3);
// Optional[Alice]

Optional<String> shortName = name.filter(n -> n.length() > 10);
// Optional.empty
```

Par exemple pour ne renvoyer un utilisateur que s'il est actif :

```java
public Optional<User> getActiveUser(Long id) {
    return userRepository.findById(id)
        .filter(User::isActive);
}
```

Ou pour valider une valeur lue dans une chaîne de caractères, ici un numéro de port :

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

Avec `"8443"`, on obtient `8443`. Avec `"abc"` ou `"70000"`, on retombe sur `8080`.

## Agir sur la valeur avec ifPresent

`ifPresent()` exécute une action seulement si la valeur est présente :

```java
Optional<User> user = findUser(id);
user.ifPresent(u -> System.out.println("User: " + u.getName()));
```

Java 9 a ajouté `ifPresentOrElse()`, qui prend en plus l'action à exécuter quand l'`Optional` est vide :

```java
user.ifPresentOrElse(
    u -> System.out.println("User: " + u.getName()),
    () -> System.out.println("User not found")
);
```

## Chercher ailleurs avec or

`or()` (Java 9) renvoie l'`Optional` s'il contient une valeur, et sinon appelle le `Supplier`, qui fournit un autre `Optional`. C'est pratique pour chercher une donnée à plusieurs endroits :

```java
Optional<User> user = findUserInCache(id)
    .or(() -> findUserInDatabase(id))
    .or(() -> findUserInBackup(id));
```

Chaque recherche n'est lancée que si les précédentes n'ont rien trouvé : si l'utilisateur est en base, `findUserInBackup()` n'est jamais appelée. Pour finir sur une exception plutôt que sur un `Optional`, il suffit d'ajouter un `.orElseThrow(...)` en bout de chaîne.

## Optional et les streams

Depuis Java 9, `stream()` transforme un `Optional` en `Stream` de zéro ou un élément. Avec `flatMap`, on ne garde que les valeurs présentes. Ici, `getEmail()` renvoie un `Optional<String>` :

```java
List<String> emails = users.stream()
    .map(User::getEmail)                    // Stream<Optional<String>>
    .flatMap(Optional::stream)              // Stream<String> (filtre les empty)
    .collect(Collectors.toList());
```

Avant Java 9, on écrivait :

```java
.filter(Optional::isPresent)
.map(Optional::get)
```

Dans l'autre sens, plusieurs opérations terminales des streams renvoient un `Optional`, comme `findFirst()`, `findAny()`, `min()` ou `max()` :

```java
Optional<User> firstActive = users.stream()
    .filter(User::isActive)
    .findFirst();
```

Les streams sont présentés en détail dans l'[introduction aux Streams en Java]({% post_url 2026-03-30-Introduction-aux-Streams-en-Java %}).

## Où ne pas utiliser Optional

La Javadoc d'`Optional` le dit clairement : il est prévu avant tout comme type de retour de méthode, quand l'absence de résultat est un cas normal et que `null` risquerait de provoquer des erreurs. Ailleurs, il complique le code plus qu'il ne l'aide.

### En paramètre de méthode

```java
// À éviter :
public void setName(Optional<String> name) {
    // ...
}
```

L'appelant doit emballer sa valeur, et rien ne l'empêche de passer `null` à la place de l'`Optional`. Une surcharge ou un paramètre annoté `@Nullable` (celui de JSpecify, par exemple) est plus simple :

```java
// Correct :
public void setName(String name) { /* ... */ }
public void setName() { /* sans nom */ }

// Ou avec annotation
public void setName(@Nullable String name) { /* ... */ }
```

### En champ de classe

```java
// À éviter :
public class User {
    private Optional<String> middleName;
}

// Correct :
public class User {
    private String middleName; // peut être null

    public Optional<String> getMiddleName() {
        return Optional.ofNullable(middleName);
    }
}
```

`Optional` n'implémente pas `Serializable` : dans une classe sérialisable, un champ de ce type fait échouer la sérialisation (`NotSerializableException: java.util.Optional`). On garde donc un champ classique, et c'est le getter qui renvoie un `Optional`.

### Comme composant d'un record

Même logique pour un [record]({% post_url 2026-01-10-Records-en-Java-simplifier-vos-DTOs %}) (Java 16+) :

```java
public record UserDTO(
    Long id,
    String name,
    Optional<String> middleName,    // À éviter
    String email
) {}

// Correct :
public record UserDTO(
    Long id,
    String name,
    String middleName,  // peut être null
    String email
) {
    // Méthode séparée pour la version Optional
    public Optional<String> middleNameOpt() {
        return Optional.ofNullable(middleName);
    }
}
```

On ne peut pas redéfinir l'accesseur `middleName()` pour qu'il renvoie un `Optional<String>` : un accesseur de record doit avoir exactement le type de son composant, sinon le compilateur refuse le code (`invalid accessor method in record`). D'où la méthode `middleNameOpt()`.

### Pour une collection

Une collection vide exprime déjà l'absence de résultat, inutile de l'emballer :

```java
// À éviter :
public Optional<List<User>> getUsers() { ... }

// Correct :
public List<User> getUsers() {
    return users != null ? users : Collections.emptyList();
}
```

### Renvoyer null à la place d'un Optional

Une méthode qui renvoie un `Optional` ne doit jamais renvoyer `null` : l'appelant qui enchaîne un `.map(...)` prendrait une `NullPointerException`, exactement ce qu'on voulait éviter. La Javadoc le précise aussi : une variable de type `Optional` ne doit jamais être `null`.

```java
// À éviter :
public Optional<User> findUser(Long id) {
    if (notFound) {
        return null;
    }
    return Optional.of(user);
}

// Correct :
public Optional<User> findUser(Long id) {
    if (notFound) {
        return Optional.empty();
    }
    return Optional.of(user);
}
```

## Optional avec Spring Data

Avec Spring Data JPA, `findById()` n'a pas besoin d'être déclarée : elle est héritée de `CrudRepository` et renvoie déjà un `Optional<T>`. Les méthodes de requête que l'on déclare peuvent aussi renvoyer un `Optional` :

```java
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
}
```

Si la requête trouve plusieurs résultats, Spring Data lance une `IncorrectResultSizeDataAccessException`.

Côté service, la chaîne `map` et `orElseThrow` donne directement le DTO ou l'exception :

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

## Et les performances ?

`Optional` n'est pas gratuit : `Optional.of()` alloue un objet à chaque appel. Pour une API, la lisibilité compte plus que cette allocation, mais dans une boucle exécutée des millions de fois, un simple test de `null` reste une option raisonnable. Pour les types primitifs, `OptionalInt`, `OptionalLong` et `OptionalDouble` évitent en plus le _boxing_.

## Voir aussi

- [Introduction aux Streams en Java]({% post_url 2026-03-30-Introduction-aux-Streams-en-Java %})
- [Records en Java : simplifier vos DTOs]({% post_url 2026-01-10-Records-en-Java-simplifier-vos-DTOs %})
- [Les Sealed classes en Java]({% post_url 2026-01-14-Sealed-classes-en-Java %})
- [Javadoc d'Optional (Java 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/Optional.html)
