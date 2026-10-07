---
layout: article
title: "Les Sealed classes en Java"
tags:
  - java
  - sealed
  - pattern-matching
author: Pierre Chopinet
---

Les sealed classes (classes scellées), finalisées en Java 17, fixent la liste exacte des classes qui ont le droit d'étendre une classe ou d'implémenter une interface. En échange, le compilateur peut vérifier qu'un `switch` traite tous les cas. Dans ce tutoriel, nous allons voir comment les déclarer et comment les combiner avec les records et le pattern matching.
<!--more-->

Dans cet article :
- Pourquoi sceller une hiérarchie
- Déclarer une classe scellée
- Les sous-types : final, sealed ou non-sealed
- Sealed classes et records
- Switch exhaustifs sans default
- Modéliser des types algébriques
- Une machine à états
- Les règles à respecter
- Sealed classes ou enum ?

Pré-requis : Java 17 ou plus récent, Java 21 pour les exemples avec `switch` et pattern matching.

## Pourquoi sceller une hiérarchie

Une interface publique peut être implémentée par n'importe quelle classe, y compris dans du code que son auteur ne connaît pas :

```java
public interface Shape {
    double area();
}

// N'importe qui peut créer une nouvelle forme
public class Hexagon implements Shape {
    public double area() { return 0; }
}
```

Le compilateur ne peut donc pas vérifier qu'un `switch` ou une suite de `if/else` sur une `Shape` couvre tous les cas, et l'auteur de l'interface ne peut pas la faire évoluer sans risquer de casser des implémentations qu'il n'a jamais vues.

Avant Java 17, les moyens de limiter l'héritage n'étaient pas satisfaisants. Une classe `final` ne peut pas être étendue du tout : il n'y a plus de hiérarchie. Une interface sans le mot-clé `public` n'est visible que dans son package, elle devient donc inutilisable par le reste du code, et rien n'empêche quelqu'un d'ajouter une classe dans ce même package.

## Déclarer une classe scellée

Une classe ou une interface déclarée `sealed` donne, avec `permits`, la liste exhaustive des sous-types autorisés :

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

L'héritage reste possible, mais uniquement pour ces trois classes. Une classe `Hexagon` qui essaierait d'implémenter `Shape` ne compilerait pas : `class is not allowed to extend sealed class: Shape (as it is not listed in its 'permits' clause)`.

## Les sous-types : final, sealed ou non-sealed

Chaque sous-type direct d'une classe scellée doit choisir ce qu'il fait de sa propre descendance, avec l'un de ces trois modificateurs :

```java
public sealed interface Vehicle permits Car, Bike, Boat {}

// final : ne peut plus être étendu
public final class Car implements Vehicle {}

// sealed : contrôle à nouveau ses propres sous-types
public sealed class Bike implements Vehicle permits MountainBike, RoadBike {}
public final class MountainBike extends Bike {}
public final class RoadBike extends Bike {}

// non-sealed : rouvre la hiérarchie (n'importe qui peut étendre)
public non-sealed class Boat implements Vehicle {}
public class Sailboat extends Boat {} // OK, hiérarchie ouverte
```

Sans l'un des trois, le compilateur refuse le sous-type avec `sealed, non-sealed or final modifiers expected`. Avec `non-sealed`, n'importe qui peut étendre `Boat`, mais `Vehicle` reste scellée : ses sous-types directs sont toujours `Car`, `Bike` et `Boat`.

## Sealed classes et records

Les records, disponibles depuis Java 16, sont implicitement `final`. Ils font donc des sous-types tout trouvés pour une hiérarchie scellée, sans rien à ajouter :

```java
public sealed interface Result<T> permits Success, Failure {}

public record Success<T>(T value) implements Result<T> {}
public record Failure<T>(String message, Throwable cause) implements Result<T> {}
```

Utilisation :

```java
public static <T> void handleResult(Result<T> result) {
    switch (result) {
        case Success<T> s -> System.out.println("Valeur : " + s.value());
        case Failure<T> f -> System.err.println("Erreur : " + f.message());
        // Pas de default nécessaire : le compilateur vérifie l'exhaustivité
    }
}
```

`handleResult(new Success<>(42))` affiche `Valeur : 42`. Pour en savoir plus sur les records, voir [Records en Java : simplifier vos DTOs]({% post_url 2026-01-10-Records-en-Java-simplifier-vos-DTOs %}).

## Switch exhaustifs sans default

Le gros intérêt d'une hiérarchie scellée apparaît dans les `switch` (Java 21). Le compilateur connaît tous les sous-types, il peut donc vérifier que chacun est traité, sans `default` :

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

Le jour où on ajoute un `record PayPal(String email)` à la liste `permits`, ce `switch` ne compile plus : `the switch statement does not cover all possible input values`. Le compilateur pointe ainsi chaque `switch` à compléter, alors qu'avec un `default` l'oubli serait passé inaperçu.

Ce contrôle a lieu à la compilation. Si la hiérarchie vient d'une bibliothèque qui ajoute un sous-type, et que le code contenant le `switch` n'est pas recompilé, l'exécution lance une `MatchException` quand le nouveau type arrive dans le `switch`. C'est le comportement prévu, décrit dans la Javadoc de `MatchException`.

### Déconstruire les records dans le switch

Avec les record patterns de Java 21, on récupère directement les composants dans le `case` :

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

`calculateArea(new Rectangle(3, 4))` renvoie `12.0`. Les record patterns sont détaillés dans l'article [Pattern matching en Java moderne]({% post_url 2025-10-23-Pattern-matching-en-Java-moderne %}).

## Modéliser des types algébriques

Une interface scellée dont chaque sous-type porte ses propres données correspond à ce qu'on appelle un type algébrique (ou *sum type*) en programmation fonctionnelle : une valeur est soit un `Circle`, soit un `Rectangle`, soit un `Triangle`, et rien d'autre.

### Un type Option

Le classique : une valeur présente ou absente. Java a déjà [Optional]({% post_url 2026-01-26-Optional-en-Java-eviter-les-NullPointerException %}) pour ça, mais l'exemple montre qu'une hiérarchie scellée peut mélanger un record et une classe classique, ici un singleton :

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

`getOrDefault(None.instance(), "Unknown")` renvoie `"Unknown"`.

### Un arbre syntaxique

Les hiérarchies scellées se prêtent bien aux structures récursives, comme l'arbre d'une expression arithmétique :

```java
public sealed interface Expr permits Constant, Add, Multiply, Variable {}
public record Constant(int value) implements Expr {}
public record Add(Expr left, Expr right) implements Expr {}
public record Multiply(Expr left, Expr right) implements Expr {}
public record Variable(String name) implements Expr {}

public static int evaluate(Expr expr, Map<String, Integer> vars) {
    return switch (expr) {
        case Constant(int n) -> n;
        case Add(var left, var right) -> evaluate(left, vars) + evaluate(right, vars);
        case Multiply(var left, var right) -> evaluate(left, vars) * evaluate(right, vars);
        case Variable(String name) -> vars.getOrDefault(name, 0);
    };
}

// Exemple : (2 + x) * 3
Expr expression = new Multiply(
    new Add(new Constant(2), new Variable("x")),
    new Constant(3)
);

int result = evaluate(expression, Map.of("x", 5)); // (2 + 5) * 3 = 21
```

Ajouter une opération (`Subtract`, `Divide`...) oblige à compléter `evaluate`, sinon le code ne compile plus.

## Une machine à états

Chaque état devient un record avec les données qui lui sont propres. Ici, le nombre de tentatives n'existe que dans l'état `Connecting`, et l'identifiant de session que dans l'état `Connected` :

```java
public sealed interface ConnectionState permits Disconnected, Connecting, Connected, Failure {}
public record Disconnected() implements ConnectionState {}
public record Connecting(int attempts) implements ConnectionState {}
public record Connected(String sessionId) implements ConnectionState {}
public record Failure(String message) implements ConnectionState {}

public class ConnectionManager {
    private ConnectionState state = new Disconnected();

    public void handleState() {
        switch (state) {
            case Disconnected() -> connect();
            case Connecting(int attempts) ->
                System.out.println("Tentative " + attempts + "...");
            case Connected(String sessionId) ->
                System.out.println("Connecté : " + sessionId);
            case Failure(String msg) ->
                System.err.println("Erreur : " + msg);
        }
    }
}
```

`Disconnected()` est un record pattern sans composant : il teste simplement le type.

## Les règles à respecter

Quelques contraintes sont vérifiées par le compilateur :
- Les sous-types doivent être dans le même module que la classe scellée, ou dans le même package si le code est dans le module sans nom, c'est-à-dire sans `module-info.java`.
- Chaque classe listée dans `permits` doit exister et étendre directement la classe scellée, sinon on obtient `invalid permits clause`.
- Chaque sous-type direct doit être `final`, `sealed` ou `non-sealed`.

Si tous les sous-types sont déclarés dans le même fichier que la classe scellée, `permits` peut être omis : le compilateur prend les sous-types de ce fichier. S'il n'en trouve aucun, il refuse la déclaration (`sealed class must have subclasses`).

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

Dans un module nommé, les sous-types peuvent être répartis dans plusieurs packages du module :

```java
// module-info.java
module com.example.shapes {
    exports com.example.shapes.api;
}

// com/example/shapes/api/Shape.java
package com.example.shapes.api;

import com.example.shapes.impl.Circle;
import com.example.shapes.impl.Rectangle;

public sealed interface Shape permits Circle, Rectangle {}

// com/example/shapes/impl/Circle.java
package com.example.shapes.impl;

import com.example.shapes.api.Shape;

public final class Circle implements Shape {}
```

Sans module nommé, ce découpage en deux packages est refusé : `class Shape in unnamed module cannot extend a sealed class in a different package`.

## Sealed classes ou enum ?

Un enum est lui aussi une liste fermée, et un `switch` sur un enum peut également se passer de `default`. La différence : chaque constante d'un enum est une instance unique, créée une fois pour toutes, alors qu'un sous-type d'une classe scellée peut avoir autant d'instances que nécessaire, chacune avec ses propres données. Pour une liste de valeurs fixes (jours de la semaine, statuts), un [enum]({% post_url 2026-04-13-Les-Enums-en-Java-bien-plus-que-des-constantes %}) suffit. Dès que chaque cas transporte des données différentes, comme `Circle(double radius)` et `Rectangle(double width, double height)`, une interface scellée avec des records est plus adaptée.

Avant Java 17, on obtenait ce genre de vérification avec le pattern Visitor, au prix de beaucoup plus de code. Attention par contre à ne pas tout sceller : une hiérarchie que d'autres développeurs doivent pouvoir étendre, comme un système de plugins, doit rester ouverte.

## Voir aussi

- [Pattern matching en Java moderne]({% post_url 2025-10-23-Pattern-matching-en-Java-moderne %})
- [Records en Java : simplifier vos DTOs]({% post_url 2026-01-10-Records-en-Java-simplifier-vos-DTOs %})
- [Les enums en Java]({% post_url 2026-04-13-Les-Enums-en-Java-bien-plus-que-des-constantes %})
- [Optional en Java : éviter les NullPointerException]({% post_url 2026-01-26-Optional-en-Java-eviter-les-NullPointerException %})
- [JEP 409 : Sealed Classes](https://openjdk.org/jeps/409)
- [Documentation Oracle sur les sealed classes](https://docs.oracle.com/en/java/javase/17/language/sealed-classes-and-interfaces.html)
- [Java Language Specification : les classes sealed, non-sealed et final](https://docs.oracle.com/javase/specs/jls/se17/html/jls-8.html#jls-8.1.1.2)
