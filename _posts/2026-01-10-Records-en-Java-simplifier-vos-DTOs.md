---
layout: article
title: "Records en Java : simplifier vos DTOs"
author: Pierre Chopinet
tags:
  - java
  - records
  - dto
---

Les records, introduits en Java 14 (preview) et finalisés en Java 16, révolutionnent l'écriture de classes de données immuables. Fini le boilerplate des getters, `equals()`, `hashCode()` et `toString()` : un record fait tout ça en une ligne.
<!--more-->

Dans cet article, vous découvrirez :
- Ce qu'est un record et pourquoi l'utiliser pour vos DTOs
- La syntaxe et les fonctionnalités des records
- Comment personnaliser les records (validation, constructeurs, méthodes)
- Les limitations et bonnes pratiques
- L'intégration avec Spring Boot, Jackson et JPA

---

## 1) Qu'est-ce qu'un record ?

Un **record** est une classe Java déclarée avec le mot-clé `record` au lieu de `class`. Il représente un agrégat immuable de données, parfait pour les DTOs (Data Transfer Objects), les valeurs métier ou les résultats de requêtes.

### Avant les records (Java ≤ 15)

```java
public final class UserDTO {
    private final Long id;
    private final String name;
    private final String email;

    public UserDTO(Long id, String name, String email) {
        this.id = id;
        this.name = name;
        this.email = email;
    }

    public Long getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        UserDTO userDTO = (UserDTO) o;
        return Objects.equals(id, userDTO.id) &&
               Objects.equals(name, userDTO.name) &&
               Objects.equals(email, userDTO.email);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, name, email);
    }

    @Override
    public String toString() {
        return "UserDTO{id=" + id + ", name='" + name + "', email='" + email + "'}";
    }
}
```

**≈ 35 lignes de code** pour une simple classe de données !

### Avec un record (Java ≥ 16)

```java
public record UserDTO(Long id, String name, String email) {}
```

**1 ligne.** Le compilateur génère automatiquement :
- Un constructeur canonique avec tous les paramètres
- Des accesseurs `id()`, `name()`, `email()` (pas de préfixe `get`)
- `equals()`, `hashCode()`, `toString()`
- La classe est `final` et les champs sont `private final`

---

## 2) Syntaxe et fonctionnalités de base

### Déclaration simple

```java
public record Point(int x, int y) {}

// Utilisation
Point p = new Point(10, 20);
System.out.println(p.x());      // 10
System.out.println(p.y());      // 20
System.out.println(p);           // Point[x=10, y=20]
```

### Immutabilité

Les composants d'un record sont `final`. Impossible de les modifier après construction.

```java
public record Product(String sku, double price) {}

Product p = new Product("ABC-123", 99.99);
// p.price = 50.0; // ERREUR : pas de setter
```

### Equals et hashCode

Basés sur **tous les composants** du record.

```java
Point p1 = new Point(5, 10);
Point p2 = new Point(5, 10);
Point p3 = new Point(5, 15);

System.out.println(p1.equals(p2));  // true
System.out.println(p1.equals(p3));  // false
```

---

## 3) Personnaliser un record

### 3.1) Validation dans le constructeur canonique

Vous pouvez ajouter de la logique de validation sans redéclarer tous les paramètres grâce au **constructeur compact**.

```java
public record Email(String address) {
    public Email {
        if (address == null || !address.contains("@")) {
            throw new IllegalArgumentException("Email invalide : " + address);
        }
        // pas besoin de `this.address = address;`, c'est automatique
    }
}

// Utilisation
Email e1 = new Email("user@example.com");  // OK
Email e2 = new Email("invalide");          // IllegalArgumentException
```

### 3.2) Constructeurs alternatifs

```java
public record Rectangle(int width, int height) {
    // Constructeur carré
    public Rectangle(int side) {
        this(side, side);
    }
}

Rectangle r1 = new Rectangle(10, 20);
Rectangle r2 = new Rectangle(15);  // Carré 15x15
```

### 3.3) Méthodes personnalisées

Un record peut contenir des méthodes métier.

```java
public record Rectangle(int width, int height) {
    public int area() {
        return width * height;
    }

    public boolean isSquare() {
        return width == height;
    }
}

Rectangle r = new Rectangle(10, 10);
System.out.println(r.area());       // 100
System.out.println(r.isSquare());   // true
```

### 3.4) Redéfinir les accesseurs

```java
public record Temperature(double celsius) {
    // Accesseur personnalisé
    @Override
    public double celsius() {
        return Math.round(celsius * 10.0) / 10.0; // arrondi à 1 décimale
    }

    public double fahrenheit() {
        return celsius * 9.0 / 5.0 + 32.0;
    }
}

Temperature t = new Temperature(23.456);
System.out.println(t.celsius());     // 23.5
System.out.println(t.fahrenheit());  // 74.3
```

---

## 4) Records et interfaces

Un record peut implémenter une ou plusieurs interfaces.

```java
public interface Identifiable {
    Long id();
}

public record UserDTO(Long id, String name, String email) implements Identifiable {}

public record ProductDTO(Long id, String sku, double price) implements Identifiable {}

// Polymorphisme
Identifiable entity = new UserDTO(1L, "Alice", "alice@example.com");
System.out.println(entity.id());  // 1
```

---

## 5) Imbrication et composition

### Records imbriqués

```java
public record Address(String street, String city, String zipCode) {}

public record Person(String name, Address address) {}

// Utilisation
Address addr = new Address("10 rue de la Paix", "Paris", "75001");
Person p = new Person("Alice", addr);
System.out.println(p.address().city());  // Paris
```

### Décomposition avec pattern matching (Java 21+)

```java
static void printCity(Person person) {
    switch (person) {
        case Person(String name, Address(var street, var city, var zip)) ->
            System.out.println(name + " habite à " + city);
    }
}
```

---

## 6) Records et collections

Les records fonctionnent parfaitement avec les collections et les streams.

```java
public record User(Long id, String name, int age) {}

List<User> users = List.of(
    new User(1L, "Alice", 30),
    new User(2L, "Bob", 25),
    new User(3L, "Charlie", 35)
);

// Filtrer et mapper
List<String> names = users.stream()
    .filter(u -> u.age() > 28)
    .map(User::name)
    .toList();
// [Alice, Charlie]

// Group by
Map<Integer, List<User>> byAge = users.stream()
    .collect(Collectors.groupingBy(User::age));
```

---

## 7) Records avec Spring Boot

### 7.1) DTOs pour les API REST

```java
public record CreateUserRequest(String name, String email) {}

public record UserResponse(Long id, String name, String email) {}

@RestController
@RequestMapping("/api/users")
public class UserController {

    @PostMapping
    public UserResponse createUser(@RequestBody @Valid CreateUserRequest request) {
        // logique de création
        return new UserResponse(1L, request.name(), request.email());
    }
}
```

### 7.2) Validation avec Bean Validation

```java
import jakarta.validation.constraints.*;

public record CreateUserRequest(
    @NotBlank(message = "Le nom est obligatoire")
    String name,

    @NotBlank @Email(message = "Email invalide")
    String email,

    @Min(18) @Max(120)
    int age
) {}
```

### 7.3) Records et Jackson (sérialisation JSON)

Jackson (depuis 2.12+) supporte nativement les records.

```java
public record Product(
    Long id,
    String name,
    @JsonProperty("unit_price") double unitPrice
) {}
```

Sérialisation/désérialisation automatique :

```json
{
  "id": 1,
  "name": "Laptop",
  "unit_price": 999.99
}
```

---

## 8) Limitations et contraintes

### Ce qu'un record **ne peut pas** faire :

- Étendre une autre classe (mais peut implémenter des interfaces)
- Avoir des champs d'instance non-finaux
- Être abstrait
- Être déclaré non-final

### Ce qu'un record **peut** faire :

- Implémenter des interfaces
- Contenir des méthodes statiques
- Contenir des champs statiques
- Être générique : `record Pair<T, U>(T first, U second) {}`
- Être imbriqué dans une classe

---

## 9) Bonnes pratiques

### ✅ Utilisez des records pour :

- **DTOs** (Request/Response dans les APIs)
- **Value Objects** (Email, Money, Coordinates)
- **Résultats de requêtes** (projections)
- **Tuples** et paires de valeurs
- **Événements** (Event Sourcing, messaging)

### ❌ N'utilisez pas de records pour :

- **Entités JPA** (utilisez des classes classiques)
- **Données mutables** (si vous avez besoin de setters)
- **Héritage complexe** (un record ne peut pas étendre une classe)

### Conseils :

- Gardez les records **simples** : pas de logique métier complexe
- Utilisez le **constructeur compact** pour les validations simples
- Préférez **composition** plutôt qu'héritage (records + interfaces)
- Documentez vos records avec Javadoc si nécessaire
- Combinez records et **pattern matching** (Java 21+) pour un code élégant

---

## 10) Records vs Lombok

Lombok offre `@Data`, `@Value` pour réduire le boilerplate. Pourquoi préférer les records ?

| Critère          | Records        | Lombok                      |
|------------------|----------------|-----------------------------|
| Standard Java    | Oui (Java 16+) | Non (dépendance externe)    |
| Compilation      | Rapide         | Annotation processor (lent) |
| IDE support      | Natif          | Plugin nécessaire           |
| Immutabilité     | Par défaut     | `@Value` seulement          |
| Lisibilité       | Excellente     | Masque le code généré       |
| Pattern matching | Oui (Java 21+) | Non                         |

**Verdict** : si vous êtes sur Java 16+, préférez les records. Lombok reste utile pour les entités JPA et certains cas complexes.

---

## Conclusion

Les records simplifient drastiquement l'écriture de classes de données en Java. En une ligne, vous obtenez une classe immuable, typée, avec `equals`, `hashCode` et `toString` générés automatiquement.

**Points clés à retenir :**

- Records = syntaxe minimale pour les données immuables
- Parfaits pour les DTOs, value objects et projections
- Personnalisables (validation, méthodes, interfaces)
- Ne conviennent pas aux entités JPA
- Natifs depuis Java 16, sans dépendances externes
- Combinables avec pattern matching (Java 21+)

Les records sont un atout majeur du Java moderne. Si vous utilisez Java 16+, adoptez-les sans hésiter pour vos DTOs !

---

## Pour aller plus loin

- [JEP 395: Records (Final, JDK 16)](https://openjdk.org/jeps/395)
- [Documentation Oracle sur les records](https://docs.oracle.com/en/java/javase/17/language/records.html)
- [Jackson support for Java records](https://github.com/FasterXML/jackson-databind/wiki/Jackson-Release-2.12#java-16-record-classes)
- [Spring Framework 6 et records](https://docs.spring.io/spring-framework/reference/core/beans/java/instantiating-container.html)

## Voir aussi

- [Pattern matching en Java moderne]({% post_url 2025-10-23-Pattern-matching-en-Java-moderne %})
- [Comment ajouter du cache à une application Spring Boot]({% post_url 2025-11-08-Comment-ajouter-du-cache-a-une-application-Spring-Boot %})
- [Introduction aux collections Java]({% post_url 2020-11-12-Framework-collections-java-intro %})
- [Comment utiliser les properties Spring]({% post_url 2021-04-29-Comment-utiliser-les-properties-spring %})
