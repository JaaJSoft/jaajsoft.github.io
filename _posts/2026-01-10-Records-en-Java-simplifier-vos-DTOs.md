---
layout: article
title: "Records en Java : simplifier vos DTOs"
description: "Les records en Java pour écrire des DTOs immuables sans boilerplate : syntaxe, personnalisation, collections, Spring Boot, limites et comparaison avec Lombok."
tags:
  - java
  - records
  - dto
author: Pierre Chopinet
---

Un DTO en Java, c'est souvent un constructeur, des _getters_, `equals()`, `hashCode()` et `toString()` : une trentaine de lignes pour trois champs. Les records, arrivés en preview avec Java 14 et finalisés en Java 16, génèrent tout ça à partir d'une seule ligne. Dans ce tutoriel, nous allons voir comment les écrire, les personnaliser, et les utiliser avec Spring Boot et Jackson.
<!--more-->

Dans cet article :
- Qu'est-ce qu'un record ?
- Utiliser un record
- Personnaliser un record
- Records imbriqués et pattern matching
- Records et collections
- Records avec Spring Boot et Jackson
- Ce qu'un record ne peut pas faire
- Records ou Lombok ?

Pré-requis : Java 16 ou plus récent, Java 21 pour les exemples avec pattern matching.

## Qu'est-ce qu'un record ?

Un record est une classe déclarée avec le mot-clé `record` au lieu de `class`. Il sert à transporter des données qui ne changent pas : le corps d'une requête ou d'une réponse d'API, une valeur métier comme un email ou un montant, le résultat d'une requête, un événement...

Voici un DTO écrit à l'ancienne, avant Java 16 :

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

35 lignes. Le même DTO avec un record :

```java
public record UserDTO(Long id, String name, String email) {}
```

À partir de cette ligne, le compilateur génère :
- un constructeur qui prend tous les composants, appelé constructeur canonique
- un accesseur par composant : `id()`, `name()` et `email()`, sans préfixe `get`
- `equals()`, `hashCode()` et `toString()`, calculés à partir de tous les composants

La classe est implicitement `final` et chaque composant devient un champ `private final`.

## Utiliser un record

Un record s'instancie comme n'importe quelle classe :

```java
public record Point(int x, int y) {}

// Utilisation
Point p = new Point(10, 20);
System.out.println(p.x());      // 10
System.out.println(p.y());      // 20
System.out.println(p);           // Point[x=10, y=20]
```

Deux records sont égaux quand tous leurs composants sont égaux :

```java
Point p1 = new Point(5, 10);
Point p2 = new Point(5, 10);
Point p3 = new Point(5, 15);

System.out.println(p1.equals(p2));  // true
System.out.println(p1.equals(p3));  // false
```

Comme `hashCode()` suit la même règle, un record fait une bonne clé de `HashMap`, par exemple pour [regrouper des données selon plusieurs critères]({% post_url 2026-01-11-Comment-faire-des-group-by-en-Java %}).

Il n'y a pas de setter et les champs sont `final`, on ne peut donc pas modifier un record après sa création :

```java
public record Product(String sku, double price) {}

Product p = new Product("ABC-123", 99.99);
// p.price = 50.0; // ne compile pas : le champ est private et final
```

Attention, cette immutabilité est superficielle (la Javadoc de `java.lang.Record` parle de *shallowly immutable*). Si un composant est une `ArrayList`, le record garde la même référence, mais le contenu de la liste peut toujours changer. On verra juste après comment faire une copie défensive.

## Personnaliser un record

### Valider dans le constructeur compact

Pour valider les valeurs, inutile de réécrire tout le constructeur. Un constructeur compact, sans liste de paramètres, s'exécute avant l'affectation des champs :

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

L'affectation des champs est ajoutée automatiquement à la fin du constructeur compact. Écrire `this.address = ...` dedans est même refusé par le compilateur (`cannot assign a value to final variable address`). Par contre, on peut réaffecter le paramètre avant cette affectation, pour faire une copie défensive :

```java
public record Team(String name, List<String> members) {
    public Team {
        members = List.copyOf(members); // copie non modifiable
    }
}
```

Avec cette copie, modifier la liste passée au constructeur n'a plus d'effet sur le record, et `team.members().add("Eve")` lance une `UnsupportedOperationException`.

Ou pour normaliser une valeur, ici un arrondi au dixième :

```java
public record Temperature(double celsius) {
    public Temperature {
        celsius = Math.round(celsius * 10.0) / 10.0; // arrondi à 1 décimale
    }

    public double fahrenheit() {
        return celsius * 9.0 / 5.0 + 32.0;
    }
}

Temperature t = new Temperature(23.456);
System.out.println(t.celsius());     // 23.5
System.out.println(t.fahrenheit());  // 74.3
```

### Ajouter des constructeurs et des méthodes

Un record peut avoir d'autres constructeurs, à condition qu'ils appellent un autre constructeur du record avec `this(...)`, et donc au bout du compte le constructeur canonique. Il peut aussi contenir des méthodes métier :

```java
public record Rectangle(int width, int height) {
    // Constructeur pour un carré
    public Rectangle(int side) {
        this(side, side);
    }

    public int area() {
        return width * height;
    }

    public boolean isSquare() {
        return width == height;
    }
}

Rectangle r = new Rectangle(10);
System.out.println(r);              // Rectangle[width=10, height=10]
System.out.println(r.area());       // 100
System.out.println(r.isSquare());   // true
```

### Redéfinir un accesseur

On peut aussi écrire soi-même un accesseur, avec exactement le même nom et le même type de retour que le composant (sinon : `invalid accessor method in record`). Par contre, il doit renvoyer la valeur du champ : la Javadoc de `java.lang.Record` impose que la copie `new Temperature(t.celsius())` soit égale à `t`. Avec l'arrondi placé dans l'accesseur `celsius()` plutôt que dans le constructeur, cette copie ne serait plus égale à l'original, et `toString()` afficherait toujours `Temperature[celsius=23.456]`.

### Implémenter une interface

Un record peut implémenter une ou plusieurs interfaces. Ici, l'accesseur généré `id()` sert directement d'implémentation à la méthode de l'interface :

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

## Records imbriqués et pattern matching

Un record peut contenir d'autres records :

```java
public record Address(String street, String city, String zipCode) {}

public record Person(String name, Address address) {}

// Utilisation
Address addr = new Address("10 rue de la Paix", "Paris", "75001");
Person p = new Person("Alice", addr);
System.out.println(p.address().city());  // Paris
```

Depuis Java 21, un record pattern permet de déstructurer un record et ceux qu'il contient en une seule fois :

```java
static void printCity(Person person) {
    switch (person) {
        case Person(String name, Address(var street, var city, var zip)) ->
            System.out.println(name + " habite à " + city);
    }
}
```

`printCity(p)` affiche `Alice habite à Paris`. Attention, le pattern imbriqué `Address(...)` ne correspond pas à une adresse `null` : avec `new Person("Bob", null)`, ce `switch` lance une `MatchException`. Depuis Java 22, les composants inutilisés comme `street` et `zip` peuvent être remplacés par `_`. Tout ça est détaillé dans l'article sur le [pattern matching en Java]({% post_url 2025-10-23-Pattern-matching-en-Java-moderne %}).

## Records et collections

Les accesseurs s'utilisent comme des références de méthode, ce qui rend les records pratiques avec les streams :

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

## Records avec Spring Boot et Jackson

### DTOs pour les API REST

Les records conviennent bien aux corps de requête et de réponse d'une API :

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

Jackson, utilisé par Spring Boot pour le JSON, désérialise le corps de la requête en appelant le constructeur canonique du record, et sérialise la réponse à partir de ses accesseurs.

### Validation avec Bean Validation

Les annotations de validation se placent directement sur les composants :

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

Pour qu'elles soient vérifiées, il faut l'annotation `@Valid` sur le paramètre du contrôleur, comme dans l'exemple précédent, et la dépendance `spring-boot-starter-validation`, qui n'est pas incluse dans `spring-boot-starter-web`.

### Renommer un champ JSON

Jackson gère les records depuis sa version 2.12, et les annotations habituelles fonctionnent sur les composants :

```java
public record Product(
    Long id,
    String name,
    @JsonProperty("unit_price") double unitPrice
) {}
```

`new Product(1L, "Laptop", 999.99)` donne le JSON suivant, et le même JSON est relu sans problème :

```json
{"id":1,"name":"Laptop","unit_price":999.99}
```

Testé avec Jackson 2.22.3 et 3.2.3. Spring Boot 4 utilise Jackson 3, dont les annotations restent dans le package `com.fasterxml.jackson.annotation` : l'exemple ne change pas.

## Ce qu'un record ne peut pas faire

Un record ne peut pas :
- hériter d'une autre classe (il hérite déjà de `java.lang.Record`), mais il peut implémenter des interfaces
- déclarer des champs d'instance en plus de ses composants (erreur `field declaration must be static`)
- être `abstract` ou être étendu, puisqu'il est implicitement `final`

Il peut par contre avoir des champs et des méthodes statiques, être générique (`record Pair<T, U>(T first, U second) {}`), et être déclaré dans une classe ou même directement dans une méthode.

Un record ne peut pas non plus servir d'entité JPA. La Javadoc de `@Entity` (Jakarta Persistence 3.2) est explicite : une entité doit être une classe non `final` avec un constructeur sans paramètre, et un record ne peut pas être désigné comme entité. Cette même version accepte par contre un record comme `@Embeddable`. Pour les entités, on garde donc des classes classiques, et les records servent aux DTOs qu'on en tire.

## Records ou Lombok ?

Lombok répond au même problème avec des annotations : `@Value` génère une classe immuable avec ses getters, `equals()`, `hashCode()` et `toString()`, et `@Data` fait la même chose pour une classe modifiable, setters compris.

La différence, c'est qu'un record fait partie du langage : pas de dépendance ni d'annotation processor à configurer, et le compilateur connaît sa structure, ce qui permet les record patterns (une classe annotée avec `@Value` ne peut pas être déstructurée dans un `switch`). Attention si vous migrez de l'un à l'autre : les accesseurs d'un record s'appellent `name()`, pas `getName()`.

Sur Java 16 ou plus, je vous conseille les records pour les DTOs et les valeurs immuables, et Lombok là où un record ne convient pas, comme les entités JPA. Les deux ne s'excluent pas : `@Builder` de Lombok fonctionne aussi sur un record (testé avec Lombok 1.18.46).

## Voir aussi

- [Pattern matching en Java moderne]({% post_url 2025-10-23-Pattern-matching-en-Java-moderne %})
- [Les Sealed classes en Java]({% post_url 2026-01-14-Sealed-classes-en-Java %})
- [Java : Comment faire des group by]({% post_url 2026-01-11-Comment-faire-des-group-by-en-Java %})
- [Introduction aux Streams en Java]({% post_url 2026-03-30-Introduction-aux-Streams-en-Java %})
- [JEP 395 : Records](https://openjdk.org/jeps/395)
- [Documentation Oracle sur les records](https://docs.oracle.com/en/java/javase/17/language/records.html)
- [Notes de version de Jackson 2.12](https://github.com/FasterXML/jackson/wiki/Jackson-Release-2.12), qui ajoute le support des records
