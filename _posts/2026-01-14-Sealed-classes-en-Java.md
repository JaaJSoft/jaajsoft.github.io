---
layout: article
title: "Les Sealed classes en Java"
author: Pierre Chopinet
tags:
  - java
  - sealed
  - pattern-matching
---

Les sealed classes (classes scellées), finalisées dans Java 17, permettent de contrôler précisément quelles classes peuvent étendre ou implémenter une classe ou interface donnée. Cette fonctionnalité offre un contrôle granulaire sur les hiérarchies de types et améliore la sûreté du code grâce à la vérification d'exhaustivité du compilateur.
<!--more-->

Dans cet article :
- Qu'est-ce qu'une sealed class et pourquoi l'utiliser
- Syntaxe et déclaration des classes scellées
- Combinaison avec les records et le pattern matching
- Modélisation de types algébriques (sum types)
- Cas d'usage pratiques
- Limitations et bonnes pratiques

Pré-requis : Java 17+ (LTS) où les sealed classes sont finalisées.

---

## 1) Problème : hiérarchies ouvertes incontrôlables

Avant les sealed classes, Java ne permettait pas de contrôler qui pouvait étendre une classe ou interface. Cette limitation posait plusieurs problèmes de maintenabilité et de sûreté du code.

### 1.1) Le problème avec les hiérarchies classiques

En Java traditionnel, une classe ou interface peut être étendue par n'importe quelle classe :

```java
public interface Shape {
    double area();
}

// N'importe qui peut créer une nouvelle forme
public class Hexagon implements Shape {
    public double area() { return 0; }
}
```

**Problèmes :**
- Impossible de garantir l'exhaustivité dans un `switch` ou `if/else`
- Les modifications de l'API peuvent casser du code client inconnu
- Difficile de raisonner sur tous les cas possibles
- Le compilateur ne peut pas aider avec des avertissements

### 1.2) Solutions traditionnelles limitées

Avant Java 17, les développeurs tentaient de contourner ce problème avec des solutions imparfaites :

**Option 1 : final (trop restrictif)**
```java
public final class Circle {
    // Impossible d'étendre, aucune hiérarchie possible
}
```

**Option 2 : package-private (contournement facile)**
```java
interface Shape { } // package-private
// Contournable en créant une classe dans le même package
```

---

## 2) Sealed classes : contrôle explicite des sous-types

Les sealed classes offrent un juste milieu : elles permettent l'héritage, mais uniquement pour un ensemble défini et contrôlé de sous-types.

### 2.1) Syntaxe de base

Voici comment déclarer une hiérarchie scellée complète :

```java
public sealed interface Shape permits Circle, Rectangle, Triangle {
    double area();
}

public final class Circle implements Shape {
    private final double radius;

    public Circle(double radius) {
        this.radius = radius;
    }

    @Override
    public double area() {
        return Math.PI * radius * radius;
    }
}

public final class Rectangle implements Shape {
    private final double width;
    private final double height;

    public Rectangle(double width, double height) {
        this.width = width;
        this.height = height;
    }

    @Override
    public double area() {
        return width * height;
    }
}

public final class Triangle implements Shape {
    private final double base;
    private final double height;

    public Triangle(double base, double height) {
        this.base = base;
        this.height = height;
    }

    @Override
    public double area() {
        return 0.5 * base * height;
    }
}
```

**Points clés :**
- `sealed` déclare que la classe/interface contrôle ses sous-types
- `permits` liste exhaustive des sous-types autorisés
- Les sous-types doivent être déclarés `final`, `sealed`, ou `non-sealed`

### 2.2) Les trois modificateurs pour les sous-types

Chaque sous-type d'une classe scellée doit explicitement déclarer son comportement vis-à-vis de l'héritage :

```java
public sealed interface Vehicle permits Car, Bike, Boat {}

// 1) final : ne peut plus être étendu
public final class Car implements Vehicle {}

// 2) sealed : contrôle à nouveau ses propres sous-types
public sealed class Bike implements Vehicle permits MountainBike, RoadBike {}
public final class MountainBike extends Bike {}
public final class RoadBike extends Bike {}

// 3) non-sealed : rouvre la hiérarchie (n'importe qui peut étendre)
public non-sealed class Boat implements Vehicle {}
public class Sailboat extends Boat {} // OK, hiérarchie ouverte
```

---

## 3) Sealed classes et records : la combinaison parfaite

Les records, introduits en Java 16, s'intègrent naturellement avec les sealed classes. Étant implicitement `final`, ils constituent des candidats idéaux pour les sous-types d'une hiérarchie scellée :



```java
public sealed interface Result<T> permits Success, Error {}

public record Success<T>(T value) implements Result<T> {}
public record Error<T>(String message, Throwable cause) implements Result<T> {}
```

Utilisation :

```java
public static <T> void handleResult(Result<T> result) {
    switch (result) {
        case Success<T> s -> System.out.println("Valeur : " + s.value());
        case Error<T> e -> System.err.println("Erreur : " + e.message());
        // Pas de default nécessaire : le compilateur vérifie l'exhaustivité
    }
}
```

---

## 4) Pattern matching exhaustif avec sealed classes

L'un des avantages majeurs des sealed classes est la vérification d'exhaustivité par le compilateur. Combinées au pattern matching de Java 21+, elles offrent une sûreté de type remarquable.

### 4.1) Switch exhaustif sans default

Le compilateur peut garantir que tous les cas sont couverts, éliminant le besoin d'un `default` :

```java
public sealed interface Payment permits CreditCard, Cash, BankTransfer {}
public record CreditCard(String number, String cvv) implements Payment {}
public record Cash(double amount) implements Payment {}
public record BankTransfer(String iban, String bic) implements Payment {}

public static void processPayment(Payment payment) {
    switch (payment) {
        case CreditCard cc -> processCreditCard(cc.number(), cc.cvv());
        case Cash cash -> processCash(cash.amount());
        case BankTransfer bt -> processBankTransfer(bt.iban(), bt.bic());
        // Pas de default : le compilateur garantit l'exhaustivité
    }
}
```

**Avantages :**
- Le compilateur vérifie que tous les cas sont couverts
- Ajouter un nouveau type `PayPal` génère des erreurs de compilation partout où il faut le gérer
- Pas de `default` qui masque des oublis

### 4.2) Déconstruction avec record patterns (Java 21+)

Avec Java 21, on peut déconstruire directement les records dans le `switch`, rendant le code encore plus expressif :

```java
public sealed interface Shape permits Circle, Rectangle, Triangle {}
public record Circle(double radius) implements Shape {}
public record Rectangle(double width, double height) implements Shape {}
public record Triangle(double base, double height) implements Shape {}

public static double calculateArea(Shape shape) {
    return switch (shape) {
        case Circle(double r) -> Math.PI * r * r;
        case Rectangle(double w, double h) -> w * h;
        case Triangle(double b, double h) -> 0.5 * b * h;
    };
}
```

---

## 5) Modéliser des types algébriques (sum types)

Les sealed classes permettent de modéliser élégamment des types algébriques, un concept bien connu en programmation fonctionnelle. Voici les patterns les plus courants.

### 5.1) Option/Maybe type

Représente une valeur qui peut être présente ou absente, alternative type-safe à `null` :

```java
public sealed interface Option<T> permits Some, None {}
public record Some<T>(T value) implements Option<T> {}
public final class None<T> implements Option<T> {
    private static final None<?> INSTANCE = new None<>();

    private None() {}

    @SuppressWarnings("unchecked")
    public static <T> None<T> instance() {
        return (None<T>) INSTANCE;
    }
}

// Utilisation
public static <T> T getOrDefault(Option<T> option, T defaultValue) {
    return switch (option) {
        case Some<T> s -> s.value();
        case None<T> n -> defaultValue;
    };
}

Option<String> name = new Some<>("Alice");
String result = getOrDefault(name, "Unknown"); // "Alice"
```

### 5.2) Either type (gauche/droite)

Représente un choix entre deux valeurs possibles, souvent utilisé pour gérer les erreurs :

```java
public sealed interface Either<L, R> permits Left, Right {}
public record Left<L, R>(L value) implements Either<L, R> {}
public record Right<L, R>(R value) implements Either<L, R> {}

// Utilisation : représenter succès ou erreur
public static Either<String, Integer> divide(int a, int b) {
    if (b == 0) {
        return new Left<>("Division par zéro");
    }
    return new Right<>(a / b);
}

Either<String, Integer> result = divide(10, 2);
switch (result) {
    case Left<String, Integer> err -> System.err.println("Erreur : " + err.value());
    case Right<String, Integer> ok -> System.out.println("Résultat : " + ok.value());
}
```

### 5.3) AST (Abstract Syntax Tree)

Les sealed classes excellent pour représenter des structures hiérarchiques comme les arbres syntaxiques abstraits :

```java
public sealed interface Expr permits Constant, Add, Multiply, Variable {}
public record Constant(int value) implements Expr {}
public record Add(Expr left, Expr right) implements Expr {}
public record Multiply(Expr left, Expr right) implements Expr {}
public record Variable(String name) implements Expr {}

public static int eval(Expr expr, Map<String, Integer> vars) {
    return switch (expr) {
        case Constant(int n) -> n;
        case Add(var left, var right) -> eval(left, vars) + eval(right, vars);
        case Multiply(var left, var right) -> eval(left, vars) * eval(right, vars);
        case Variable(String name) -> vars.getOrDefault(name, 0);
    };
}

// Exemple : (2 + x) * 3
Expr expression = new Multiply(
    new Add(new Constant(2), new Variable("x")),
    new Constant(3)
);

int result = eval(expression, Map.of("x", 5)); // (2 + 5) * 3 = 21
```

---

## 6) Cas d'usage pratiques

Explorons quelques exemples concrets où les sealed classes apportent une vraie valeur ajoutée dans le code métier.

### 6.1) Modéliser un état d'application

Les machines à états se modélisent naturellement avec des sealed classes :

```java
public sealed interface ConnectionState permits Disconnected, Connecting, Connected, Error {}
public record Disconnected() implements ConnectionState {}
public record Connecting(int attempts) implements ConnectionState {}
public record Connected(String sessionId) implements ConnectionState {}
public record Error(String message) implements ConnectionState {}

public class ConnectionManager {
    private ConnectionState state = new Disconnected();

    public void handleState() {
        switch (state) {
            case Disconnected() -> connect();
            case Connecting(int attempts) ->
                System.out.println("Tentative " + attempts + "...");
            case Connected(String sessionId) ->
                System.out.println("Connecté : " + sessionId);
            case Error(String msg) ->
                System.err.println("Erreur : " + msg);
        }
    }
}
```

### 6.2) Événements dans un système

Pour les architectures événementielles, les sealed classes garantissent que tous les types d'événements sont gérés :

```java
public sealed interface Event permits UserRegistered, OrderPlaced, PaymentProcessed {}
public record UserRegistered(String userId, String email) implements Event {}
public record OrderPlaced(String orderId, String userId, double amount) implements Event {}
public record PaymentProcessed(String paymentId, String orderId, boolean success) implements Event {}

public class EventHandler {
    public void handle(Event event) {
        switch (event) {
            case UserRegistered(var userId, var email) ->
                sendWelcomeEmail(email);
            case OrderPlaced(var orderId, var userId, var amount) ->
                processOrder(orderId, userId, amount);
            case PaymentProcessed(var paymentId, var orderId, var success) ->
                updateOrderStatus(orderId, success);
        }
    }
}
```

### 6.3) Réponses HTTP typées

Modéliser les différentes réponses d'une API de manière type-safe :

```java
public sealed interface ApiResponse<T> permits Success, ClientError, ServerError {}
public record Success<T>(T data, int statusCode) implements ApiResponse<T> {}
public record ClientError<T>(String message, int statusCode) implements ApiResponse<T> {}
public record ServerError<T>(String message, int statusCode, Throwable cause) implements ApiResponse<T> {}

public static <T> void handleResponse(ApiResponse<T> response) {
    switch (response) {
        case Success<T> s ->
            System.out.println("Données : " + s.data());
        case ClientError<T> e ->
            System.err.println("Erreur client (" + e.statusCode() + ") : " + e.message());
        case ServerError<T> e ->
            System.err.println("Erreur serveur (" + e.statusCode() + ") : " + e.message());
    }
}
```

---

## 7) Règles et contraintes

Les sealed classes sont soumises à plusieurs règles strictes pour garantir leur cohérence et leur sûreté.

### 7.1) Règles de base

1 **Les sous-types doivent être accessibles** à la classe scellée (même package ou module)

2 **Déclaration explicite requise** : tous les sous-types listés dans `permits` doivent exister

3 **Sous-types dans le même fichier** : si tous les sous-types sont dans le même fichier, `permits` peut être omis (inféré)
```java
// Fichier Shape.java
public sealed interface Shape {
    double area();
}

// Dans le même fichier, permits est optionnel
final class Circle implements Shape {
    public double area() { return 0; }
}

final class Rectangle implements Shape {
    public double area() { return 0; }
}
```
4 **Chaque sous-type doit choisir** : `final`, `sealed`, ou `non-sealed`

### 7.2) Contraintes avec les modules

Le système de modules Java (JPMS) s'intègre avec les sealed classes pour un contrôle encore plus fin :



```java
// module-info.java
module com.example.shapes {
    exports com.example.shapes.api;
}

// com.example.shapes.api.Shape
public sealed interface Shape permits Circle, Rectangle { }

// com.example.shapes.impl.Circle
public final class Circle implements Shape { }
```

---

## 8) Sealed classes vs alternatives

Comparons les sealed classes aux autres approches pour contrôler l'héritage en Java :

| Approche               | Avantages                          | Inconvénients                              |
|------------------------|------------------------------------|--------------------------------------------|
| **Sealed classes**     | Exhaustivité, sécurité, évolutif   | Java 17+ requis                            |
| **Enum**               | Simple, exhaustif                  | Pas de données associées riches            |
| **final class**        | Empêche héritage                   | Pas de hiérarchie possible                 |
| **package-private**    | Limite la portée                   | Facilement contournable                    |
| **Visitor pattern**    | Extensible                         | Verbeux, complexe                          |

**Quand utiliser sealed classes :**
- Hiérarchies de types fermées et bien définies
- Besoin de vérification d'exhaustivité
- Modélisation de types algébriques
- APIs publiques nécessitant un contrôle strict

**Quand utiliser autre chose :**
- Enum : types simples sans données complexes
- Classes ouvertes : hiérarchies extensibles par les utilisateurs
- Interfaces : contrats flexibles sans contrôle des implémentations

---

## 9) Bonnes pratiques

Pour tirer le meilleur parti des sealed classes, voici les recommandations et pièges à éviter.

### ✅ À faire

1 **Combiner avec records** pour des hiérarchies concises et immuables

```java
public sealed interface Message permits TextMessage, ImageMessage {}
public record TextMessage(String content) implements Message {}
public record ImageMessage(String url, int width, int height) implements Message {}
```

2 **Utiliser pour modéliser des états** ou des résultats d'opérations

3 **Documenter l'intention** : expliquer pourquoi la hiérarchie est fermée

4 **Profiter de l'exhaustivité** : éviter les `default` inutiles dans les switch

5 **Nommer clairement** les sous-types pour refléter leur rôle

### ❌ À éviter

1 **Ne pas sceller systématiquement** : les hiérarchies extensibles ont leur place

2 **Éviter trop de niveaux** : sealed → sealed → sealed devient complexe

3 **Ne pas mélanger sealed et non-sealed** sans raison claire

4 **Attention aux dépendances cycliques** entre sealed types

---

## Conclusion

Les sealed classes apportent un contrôle précis sur les hiérarchies de types en Java, comblant un vide entre les classes finales (trop restrictives) et les hiérarchies ouvertes (trop permissives).

**Points clés :**
- **Contrôle explicite** des sous-types avec `permits`
- **Exhaustivité** vérifiée par le compilateur dans les switch
- **Combinaison puissante** avec records et pattern matching
- **Modélisation claire** de types algébriques (Option, Either, AST)
- **Évolution sûre** : ajout d'un sous-type = erreurs de compilation explicites

Finalisées en Java 17 LTS, les sealed classes sont un outil essentiel du Java moderne pour écrire du code type-safe et maintenable.

---

## Pour aller plus loin

- [JEP 409: Sealed Classes (Final, JDK 17)](https://openjdk.org/jeps/409)
- [Documentation Oracle sur les sealed classes](https://docs.oracle.com/en/java/javase/17/language/sealed-classes-and-interfaces.html)
- [Java Language Specification - Sealed Classes](https://docs.oracle.com/javase/specs/jls/se17/html/jls-8.html#jls-8.1.1.2)

## Voir aussi

- [Pattern matching en Java moderne]({% post_url 2025-10-23-Pattern-matching-en-Java-moderne %})
- [Records en Java : simplifier vos DTOs]({% post_url 2026-01-10-Records-en-Java-simplifier-vos-DTOs %})
- [Introduction aux collections Java]({% post_url 2020-11-12-Framework-collections-java-intro %})
- [Comment faire des group by en Java]({% post_url 2026-01-11-Comment-faire-des-group-by-en-Java %})
