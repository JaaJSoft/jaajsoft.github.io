---
layout: article
title: "Les enums en Java"
author: Pierre Chopinet
tags:
  - java
  - enum
---

Un enum représente un ensemble fixe de valeurs : les saisons, les statuts d'une commande, des niveaux de priorité... En Java, chaque constante d'un enum est un objet, qui peut avoir ses propres champs et méthodes. Dans ce tutoriel, nous allons voir comment déclarer et utiliser un enum, comment lui ajouter des données et du comportement, puis comment se servir des collections `EnumSet` et `EnumMap`.
<!--more-->

Dans cet article :
- Avant les enums : des constantes int
- Déclarer un enum
- Les enums dans un switch
- Ajouter des champs et un constructeur
- Implémenter une interface
- Une méthode différente pour chaque constante
- EnumSet
- EnumMap
- Une machine à états
- Retrouver une constante à partir d'une valeur

Pré-requis : les enums existent depuis Java 5, mais certains exemples utilisent les switch expressions (Java 14). Les exemples ont été testés avec Java 21.

## Avant les enums : des constantes int

Avant les enums, on représentait un ensemble fini de valeurs avec des constantes `int` ou `String` :

```java
public class OrderStatus {
    public static final int PENDING = 0;
    public static final int CONFIRMED = 1;
    public static final int SHIPPED = 2;
    public static final int DELIVERED = 3;
}

public void process(int status) {
    if (status == OrderStatus.CONFIRMED) {
        // ...
    }
}

// Rien n'empêche de passer n'importe quel int
process(42); // compile sans erreur !
process(-1); // aucun avertissement
```

Le compilateur ne peut rien vérifier : n'importe quel `int` est accepté. Dans les logs, on lit `status=2`, et il faut aller voir la classe pour savoir de quoi il s'agit. Rien n'empêche non plus de comparer un statut de commande avec une constante d'une autre classe qui vaut aussi 2, et il est impossible d'associer un comportement à une valeur.

## Déclarer un enum

Un enum définit un type qui n'accepte qu'un ensemble fixe de constantes nommées :

```java
public enum Season {
    SPRING, SUMMER, AUTUMN, WINTER
}
```

Chaque constante connaît son nom et sa position dans la déclaration :

```java
Season s = Season.SUMMER;

System.out.println(s);            // SUMMER
System.out.println(s.name());     // SUMMER
System.out.println(s.ordinal());  // 1 (position dans la déclaration)
```

Une méthode qui attend une `Season` n'accepte qu'une `Season` :

```java
public void plan(Season season) {
    // Seules les 4 saisons sont acceptées
}

plan(Season.SPRING); // OK
// plan(42);          // ERREUR de compilation
// plan("SPRING");    // ERREUR de compilation
```

Les deux appels commentés sont refusés par le compilateur, avec l'erreur `incompatible types: int cannot be converted to Season` pour le premier.

Chaque constante n'existe qu'en un seul exemplaire, on peut donc comparer des enums avec `==`, sans passer par `equals()` :

```java
Season s = Season.WINTER;

if (s == Season.WINTER) {
    System.out.println("Il fait froid !");
}
```

`valueOf()` retrouve une constante à partir de son nom. Elle est sensible à la casse, et lève une exception si le nom n'existe pas :

```java
Season s = Season.valueOf("SUMMER"); // Season.SUMMER
Season x = Season.valueOf("RAIN");   // IllegalArgumentException: No enum constant Season.RAIN
```

`Season.valueOf("summer")` échoue de la même façon. Enfin, `values()` renvoie toutes les constantes, dans l'ordre de leur déclaration :

```java
for (Season s : Season.values()) {
    System.out.println(s);
}
// SPRING
// SUMMER
// AUTUMN
// WINTER
```

Attention à `ordinal()` : la valeur change dès qu'on ajoute une constante au milieu de l'enum ou qu'on réordonne les constantes. Il ne faut donc pas s'en servir dans la logique métier, ni stocker cette valeur en base de données. La Javadoc précise d'ailleurs que la plupart des développeurs n'en auront pas l'usage : la méthode est surtout destinée à `EnumSet` et `EnumMap`. Pour associer une valeur stable à chaque constante, on utilise un champ (voir plus bas).

## Les enums dans un switch

Les enums s'utilisent naturellement dans un `switch`. Avec une switch expression (Java 14+), le compilateur vérifie que toutes les constantes sont traitées, pas besoin de `default` :

```java
public static String describe(Season season) {
    return switch (season) {
        case SPRING -> "Les fleurs poussent";
        case SUMMER -> "Il fait chaud";
        case AUTUMN -> "Les feuilles tombent";
        case WINTER -> "Il neige";
    };
}
```

Si on oublie `WINTER`, la compilation échoue avec `the switch expression does not cover all possible input values`. C'est un vrai avantage le jour où on ajoute une constante : le compilateur indique tous les `switch` à compléter.

Attention, cette vérification ne concerne que les switch expressions. Un `switch` utilisé comme instruction (qui ne renvoie pas de valeur), avec des `case X:` ou des `case X ->`, peut ignorer des constantes sans que le compilateur ne dise rien : pour `WINTER`, il ne fait simplement rien.

## Ajouter des champs et un constructeur

Chaque constante peut porter des données, passées à un constructeur. L'exemple classique est celui des planètes, avec leur masse et leur rayon :

```java
public enum Planet {
    MERCURY(3.303e+23, 2.4397e6),
    VENUS  (4.869e+24, 6.0518e6),
    EARTH  (5.976e+24, 6.37814e6),
    MARS   (6.421e+23, 3.3972e6),
    JUPITER(1.9e+27,   7.1492e7),
    SATURN (5.688e+26, 6.0268e7),
    URANUS (8.686e+25, 2.5559e7),
    NEPTUNE(1.024e+26, 2.4746e7);

    private final double mass;    // en kg
    private final double radius;  // en mètres

    Planet(double mass, double radius) {
        this.mass = mass;
        this.radius = radius;
    }

    // Constante gravitationnelle
    private static final double G = 6.67300E-11;

    public double surfaceGravity() {
        return G * mass / (radius * radius);
    }

    public double surfaceWeight(double otherMass) {
        return otherMass * surfaceGravity();
    }
}
```

Le poids est une force : la masse multipliée par la gravité de surface, en newtons. Pour une personne de 75 kg :

```java
double masse = 75.0;  // en kg

for (Planet p : Planet.values()) {
    System.out.printf("Poids sur %s : %.1f N%n", p, p.surfaceWeight(masse));
}
```

```
Poids sur MERCURY : 277.7 N
Poids sur VENUS : 665.4 N
Poids sur EARTH : 735.2 N
Poids sur MARS : 278.4 N
Poids sur JUPITER : 1860.5 N
Poids sur SATURN : 783.7 N
Poids sur URANUS : 665.4 N
Poids sur NEPTUNE : 836.9 N
```

Le constructeur d'un enum est toujours privé, même sans le mot-clé `private` : seules les constantes déclarées dans l'enum peuvent l'appeler. Les champs sont en général `final`. Une constante est partagée par toute l'application, la modifier reviendrait à modifier une variable globale.

## Implémenter une interface

Un enum peut implémenter une interface, et s'utiliser partout où cette interface est attendue :

```java
public interface Printable {
    String toPrettyString();
}

public enum Priority implements Printable {
    LOW, MEDIUM, HIGH, CRITICAL;

    @Override
    public String toPrettyString() {
        return name().charAt(0) + name().substring(1).toLowerCase();
    }
}

Printable p = Priority.HIGH;
System.out.println(p.toPrettyString()); // High
```

## Une méthode différente pour chaque constante

Une méthode abstraite déclarée dans l'enum doit être implémentée par chaque constante. On associe ainsi un comportement différent à chaque valeur :

```java
public enum Operation {
    ADD {
        @Override
        public double apply(double a, double b) { return a + b; }
    },
    SUBTRACT {
        @Override
        public double apply(double a, double b) { return a - b; }
    },
    MULTIPLY {
        @Override
        public double apply(double a, double b) { return a * b; }
    },
    DIVIDE {
        @Override
        public double apply(double a, double b) {
            if (b == 0) throw new ArithmeticException("Division par zéro");
            return a / b;
        }
    };

    public abstract double apply(double a, double b);
}
```

```java
double result = Operation.ADD.apply(10, 3);       // 13.0
double result2 = Operation.MULTIPLY.apply(4, 5);  // 20.0

// Parcourir toutes les opérations
for (Operation op : Operation.values()) {
    System.out.printf("%.0f %s %.0f = %.2f%n", 10.0, op, 3.0, op.apply(10, 3));
}
```

```
10 ADD 3 = 13.00
10 SUBTRACT 3 = 7.00
10 MULTIPLY 3 = 30.00
10 DIVIDE 3 = 3.33
```

Ajouter une constante sans implémenter `apply` ne compile pas (`Operation is abstract; cannot be instantiated`), aucun cas ne peut donc être oublié. Par contre, si le code est presque le même pour toutes les constantes, une seule méthode avec un `switch` reste plus lisible.

Petit détail à connaître : une constante qui a son propre corps de classe est une sous-classe anonyme de l'enum. `Operation.ADD.getClass()` renvoie donc `class Operation$1`, et c'est `getDeclaringClass()` qui renvoie `Operation`.

## EnumSet

`EnumSet` est une implémentation de `Set` réservée aux enums. En interne, c'est un vecteur de bits : un simple `long` tant que l'enum a 64 constantes ou moins. Toutes les opérations de base se font en temps constant, et sont en général bien plus rapides qu'avec un `HashSet`. Les éléments sont toujours parcourus dans l'ordre de déclaration.

```java
public enum Permission {
    READ, WRITE, EXECUTE, DELETE
}

// Créer un EnumSet
EnumSet<Permission> readOnly = EnumSet.of(Permission.READ);
EnumSet<Permission> readWrite = EnumSet.of(Permission.READ, Permission.WRITE);
EnumSet<Permission> all = EnumSet.allOf(Permission.class);   // [READ, WRITE, EXECUTE, DELETE]
EnumSet<Permission> none = EnumSet.noneOf(Permission.class); // []

// Opérations
readWrite.add(Permission.EXECUTE);
readWrite.contains(Permission.WRITE); // true

// Complémentaire
EnumSet<Permission> notReadOnly = EnumSet.complementOf(readOnly);
// [WRITE, EXECUTE, DELETE]

// Intervalle
EnumSet<Permission> range = EnumSet.range(Permission.READ, Permission.EXECUTE);
// [READ, WRITE, EXECUTE]
```

Pour associer un ensemble de permissions à chaque rôle, on passe l'`EnumSet` au constructeur de l'enum :

```java
public enum Role {
    VIEWER(EnumSet.of(Permission.READ)),
    EDITOR(EnumSet.of(Permission.READ, Permission.WRITE)),
    MODERATOR(EnumSet.of(Permission.READ, Permission.WRITE, Permission.DELETE)),
    ADMIN(EnumSet.allOf(Permission.class));

    private final EnumSet<Permission> permissions;

    Role(EnumSet<Permission> permissions) {
        this.permissions = permissions;
    }

    public boolean hasPermission(Permission permission) {
        return permissions.contains(permission);
    }
}

// Utilisation
Role.EDITOR.hasPermission(Permission.READ);    // true
Role.EDITOR.hasPermission(Permission.DELETE);  // false
Role.ADMIN.hasPermission(Permission.DELETE);   // true
```

Remplir les permissions depuis un bloc `static`, avec `VIEWER.permissions = EnumSet.of(...)`, ne fonctionne pas : un champ `final` ne peut être affecté que dans le constructeur, et le compilateur refuse le code avec l'erreur `cannot assign a value to final variable permissions`.

## EnumMap

`EnumMap` est l'équivalent de `EnumSet` pour les `Map` dont les clés sont des constantes d'enum. En interne, c'est un tableau indexé par l'ordinal de la clé, plus compact qu'une `HashMap`, et en général plus rapide :

```java
public enum Day {
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY
}

EnumMap<Day, String> schedule = new EnumMap<>(Day.class);
schedule.put(Day.FRIDAY, "Démo sprint");
schedule.put(Day.MONDAY, "Réunion d'équipe");
schedule.put(Day.WEDNESDAY, "Code review");

schedule.forEach((day, task) ->
    System.out.println(day + " : " + task)
);
```

```
MONDAY : Réunion d'équipe
WEDNESDAY : Code review
FRIDAY : Démo sprint
```

Les entrées sortent dans l'ordre de déclaration des constantes, quel que soit l'ordre d'insertion, alors que l'ordre d'une `HashMap` n'est pas garanti. Autre différence : une `EnumMap` refuse les clés `null` (`NullPointerException` au `put`), là où une `HashMap` les accepte.

## Une machine à états

Avec une méthode différente par constante, un enum peut aussi décrire une machine à états, dont chaque état connaît le suivant :

```java
public enum OrderState {
    CREATED {
        @Override
        public OrderState next() { return PAID; }
    },
    PAID {
        @Override
        public OrderState next() { return SHIPPED; }
    },
    SHIPPED {
        @Override
        public OrderState next() { return DELIVERED; }
    },
    DELIVERED {
        @Override
        public OrderState next() { return this; } // état final
    },
    CANCELLED {
        @Override
        public OrderState next() { return this; } // état final
    };

    public abstract OrderState next();

    public boolean isFinal() {
        return this == DELIVERED || this == CANCELLED;
    }
}

// Utilisation
OrderState state = OrderState.CREATED;
while (!state.isFinal()) {
    System.out.println(state + " -> " + state.next());
    state = state.next();
}
// CREATED -> PAID
// PAID -> SHIPPED
// SHIPPED -> DELIVERED
```

Les transitions sont toutes au même endroit, et il est impossible de passer dans un état qui n'existe pas.

## Retrouver une constante à partir d'une valeur

On a souvent besoin de convertir une valeur externe (un code en base de données, un champ d'une API, un symbole...) en constante. `valueOf()` ne fonctionne qu'avec le nom de la constante, il faut donc écrire sa propre méthode de recherche. Le plus simple est de construire une `Map` une fois pour toutes :

```java
public enum Currency {
    EUR("Euro", "€"),
    USD("US Dollar", "$"),
    GBP("British Pound", "£"),
    JPY("Japanese Yen", "¥");

    private final String displayName;
    private final String symbol;

    Currency(String displayName, String symbol) {
        this.displayName = displayName;
        this.symbol = symbol;
    }

    public String displayName() { return displayName; }
    public String symbol() { return symbol; }

    // Lookup par symbole (cache statique)
    private static final Map<String, Currency> BY_SYMBOL =
        Arrays.stream(values())
              .collect(Collectors.toMap(Currency::symbol, c -> c));

    public static Optional<Currency> fromSymbol(String symbol) {
        return Optional.ofNullable(BY_SYMBOL.get(symbol));
    }
}

// Utilisation
Currency.fromSymbol("€").ifPresent(c ->
    System.out.println(c.displayName()) // Euro
);

Currency.fromSymbol("?"); // Optional.empty
```

On pourrait être tenté de remplir la `Map` directement dans le constructeur, avec un `BY_SYMBOL.put(symbol, this)`. Le compilateur le refuse (`illegal reference to static field from initializer`) : les constantes sont créées en premier, avant l'initialisation des autres champs statiques de l'enum, et la `Map` n'existerait pas encore au moment où le constructeur s'exécute. À l'inverse, quand `BY_SYMBOL` est initialisée, toutes les constantes existent déjà, `values()` peut donc servir à la remplir.

## Voir aussi

- [Les Sealed classes en Java]({% post_url 2026-01-14-Sealed-classes-en-Java %})
- [Pattern matching en Java moderne]({% post_url 2025-10-23-Pattern-matching-en-Java-moderne %})
- [Les ensembles (Set) en Java]({% post_url 2025-09-25-Framework-collections-java-set %})
- [Les maps (Map) en Java]({% post_url 2025-10-04-Framework-collections-java-map %})
- [Java Language Specification : les enums](https://docs.oracle.com/javase/specs/jls/se21/html/jls-8.html#jls-8.9)
- [Effective Java, Item 34 : Use enums instead of int constants](https://www.oreilly.com/library/view/effective-java/9780134686097/)
- [Javadoc de EnumSet](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/EnumSet.html)
- [Javadoc de EnumMap](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/EnumMap.html)
