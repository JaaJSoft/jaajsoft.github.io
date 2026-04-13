---
layout: article
title: "Les Enums en Java : bien plus que des constantes"
author: Pierre Chopinet
tags:
  - java
  - enum
---

Les enums en Java sont bien plus puissantes que de simples constantes. Contrairement aux enums d'autres langages, celles de Java sont de véritables classes : elles peuvent contenir des champs, des méthodes, implémenter des interfaces, et même porter de la logique métier.
<!--more-->

Dans cet article :
- Qu'est-ce qu'un enum et pourquoi les utiliser
- Ajouter des champs, constructeurs et méthodes
- Implémenter des interfaces et de la logique par constante
- Les collections spécialisées `EnumSet` et `EnumMap`
- Cas d'usage pratiques et bonnes pratiques

Pré-requis : Java 8+ pour les bases, Java 21+ pour les exemples avec pattern matching.

---

## Le problème : les constantes magiques

Avant les enums, les développeurs utilisaient des constantes `int` ou `String` pour représenter un ensemble fini de valeurs. Cette approche classique pose plusieurs problèmes de sûreté et de lisibilité.

### Avec des constantes int

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

**Problèmes :**
- Aucune vérification de type : n'importe quel `int` est accepté
- Pas de lisibilité dans les logs : `status=2` ne dit rien
- Pas de namespace : risque de collision entre constantes
- Impossible d'ajouter du comportement aux valeurs

---

## Déclarer un enum

Un enum définit un type avec un ensemble fixe de constantes nommées.

```java
public enum Season {
    SPRING, SUMMER, AUTUMN, WINTER
}
```

Cette simple déclaration apporte déjà beaucoup par rapport aux constantes :

```java
Season s = Season.SUMMER;

System.out.println(s);            // SUMMER
System.out.println(s.name());     // SUMMER
System.out.println(s.ordinal());  // 1 (position dans la déclaration)
```

### Type safety

Le compilateur empêche les valeurs invalides :

```java
public void plan(Season season) {
    // Seules les 4 saisons sont acceptées
}

plan(Season.SPRING); // OK
// plan(42);          // ERREUR de compilation
// plan("SPRING");    // ERREUR de compilation
```

### Comparaison

Les enums se comparent avec `==` (pas besoin de `equals()`) car chaque constante est un singleton :

```java
Season s = Season.WINTER;

if (s == Season.WINTER) {
    System.out.println("Il fait froid !");
}
```

### Conversion depuis une String

La méthode `valueOf()` convertit une chaîne en constante enum :

```java
Season s = Season.valueOf("SUMMER"); // Season.SUMMER
Season x = Season.valueOf("RAIN");   // IllegalArgumentException
```

### Itérer sur les valeurs

La méthode `values()` retourne un tableau de toutes les constantes :

```java
for (Season s : Season.values()) {
    System.out.println(s);
}
// SPRING
// SUMMER
// AUTUMN
// WINTER
```

---

## Utiliser un enum dans un switch

Les enums s'intègrent naturellement avec le `switch` :

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

Le compilateur vérifie l'exhaustivité : si vous oubliez un cas, vous obtenez une erreur de compilation. Pas besoin de `default`.

---

## Ajouter des champs et un constructeur

C'est là que les enums Java se distinguent des autres langages. Chaque constante peut porter des données :

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

Utilisation :

```java
double earthWeight = 75.0;
double mass = earthWeight / Planet.EARTH.surfaceGravity();

for (Planet p : Planet.values()) {
    System.out.printf("Votre poids sur %s : %.2f N%n", p, p.surfaceWeight(mass));
}
```

**Points clés :**
- Le constructeur est toujours `private` (implicitement)
- Les champs peuvent être `final` pour l'immutabilité
- Chaque constante est une instance unique de l'enum

---

## Implémenter une interface

Les enums peuvent implémenter des interfaces, ce qui les rend compatibles avec le polymorphisme :

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

---

## Méthodes spécifiques par constante

Chaque constante peut redéfinir une méthode abstraite, ce qui permet d'associer un comportement différent à chaque valeur :

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

Utilisation :

```java
double result = Operation.ADD.apply(10, 3);      // 13.0
double result2 = Operation.MULTIPLY.apply(4, 5);  // 20.0

// Parcourir toutes les opérations
for (Operation op : Operation.values()) {
    System.out.printf("%.0f %s %.0f = %.2f%n", 10.0, op, 3.0, op.apply(10, 3));
}
// 10 ADD 3 = 13.00
// 10 SUBTRACT 3 = 7.00
// 10 MULTIPLY 3 = 30.00
// 10 DIVIDE 3 = 3.33
```

---

## EnumSet : le Set optimisé pour les enums

`EnumSet` est une implémentation de `Set` spécialement conçue pour les enums. En interne, il utilise un bitmask, ce qui le rend extrêmement rapide et économe en mémoire.

```java
public enum Permission {
    READ, WRITE, EXECUTE, DELETE
}

// Créer un EnumSet
EnumSet<Permission> readOnly = EnumSet.of(Permission.READ);
EnumSet<Permission> readWrite = EnumSet.of(Permission.READ, Permission.WRITE);
EnumSet<Permission> all = EnumSet.allOf(Permission.class);
EnumSet<Permission> none = EnumSet.noneOf(Permission.class);

// Opérations
readWrite.add(Permission.EXECUTE);
readWrite.contains(Permission.WRITE); // true

// Complémentaire
EnumSet<Permission> notReadOnly = EnumSet.complementOf(readOnly);
// [WRITE, EXECUTE, DELETE]

// Range
EnumSet<Permission> range = EnumSet.range(Permission.READ, Permission.EXECUTE);
// [READ, WRITE, EXECUTE]
```

### Exemple : gestion de rôles

```java
public enum Role {
    VIEWER, EDITOR, MODERATOR, ADMIN;

    private final EnumSet<Permission> permissions;

    static {
        VIEWER.permissions = EnumSet.of(Permission.READ);
        EDITOR.permissions = EnumSet.of(Permission.READ, Permission.WRITE);
        MODERATOR.permissions = EnumSet.of(Permission.READ, Permission.WRITE, Permission.DELETE);
        ADMIN.permissions = EnumSet.allOf(Permission.class);
    }

    // On ne peut pas utiliser un bloc static pour initialiser des champs d'instance
    // dans un enum de cette façon. Voici la version correcte :
}
```

Une approche plus idiomatique :

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
Role.EDITOR.hasPermission(Permission.DELETE);   // false
Role.ADMIN.hasPermission(Permission.DELETE);    // true
```

---

## EnumMap : le Map optimisé pour les clés enum

`EnumMap` est l'équivalent de `EnumSet` pour les `Map`. Il utilise un tableau interne indexé par ordinal, ce qui le rend plus rapide et plus compact qu'un `HashMap`.

```java
public enum Day {
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY
}

EnumMap<Day, String> schedule = new EnumMap<>(Day.class);
schedule.put(Day.MONDAY, "Réunion d'équipe");
schedule.put(Day.WEDNESDAY, "Code review");
schedule.put(Day.FRIDAY, "Démo sprint");

// Itération (toujours dans l'ordre de déclaration)
schedule.forEach((day, task) ->
    System.out.println(day + " : " + task)
);
```

### Pourquoi préférer EnumMap à HashMap ?

| Critère | `EnumMap` | `HashMap` |
|---------|-----------|-----------|
| Performance | O(1) par tableau | O(1) amorti par hash |
| Mémoire | Tableau compact | Table de hachage + entries |
| Ordre d'itération | Ordre de déclaration | Non garanti |
| Null keys | Non autorisé | Autorisé |

---

## Enum et pattern matching (Java 21+)

Depuis Java 21, les enums s'intègrent avec le pattern matching amélioré et les guarded patterns :

```java
public enum HttpStatus {
    OK(200), NOT_FOUND(404), INTERNAL_ERROR(500);

    private final int code;

    HttpStatus(int code) {
        this.code = code;
    }

    public int code() {
        return code;
    }
}

public static String categorize(HttpStatus status) {
    return switch (status) {
        case OK -> "Succès";
        case NOT_FOUND -> "Ressource introuvable";
        case INTERNAL_ERROR -> "Erreur serveur";
    };
}
```

---

## Cas d'usage : machine à états

Les enums sont idéales pour modéliser des machines à états avec des transitions contrôlées :

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

---

## Cas d'usage : conversion et parsing

Un pattern fréquent consiste à mapper des valeurs externes (base de données, API, fichiers) vers des constantes enum :

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

Currency.fromSymbol("?"); // Optional.empty()
```
---

## Bonnes pratiques

### A faire

- **Utiliser des enums** plutôt que des constantes `int` ou `String` pour les ensembles finis
- **Préférer `EnumSet` et `EnumMap`** aux `HashSet` et `HashMap` quand la clé est un enum
- **Nommer les constantes en UPPER_SNAKE_CASE** par convention Java
- **Ajouter des champs** quand les constantes portent des métadonnées (label, code, symbole)
- **Créer un cache statique** (`Map`) pour les lookups personnalisés (par code, label, etc.)

### A ne pas faire

- **Éviter `ordinal()`** pour de la logique métier : l'ajout d'une constante décale les valeurs
- **Ne pas abuser des méthodes abstraites** : si la logique est identique pour presque toutes les constantes, préférez une méthode avec `switch`
- **Éviter les enums mutables** : gardez les champs `final`

---

## Conclusion

Les enums en Java vont bien au-delà de simples constantes nommées. Avec des champs, des constructeurs, des méthodes et la possibilité d'implémenter des interfaces, elles constituent un outil puissant pour modéliser des types finis avec du comportement associé.

**Points clés :**
- **Type safety** : le compilateur vérifie les valeurs à la compilation
- **Données enrichies** : champs, constructeurs et méthodes par constante
- **Collections optimisées** : `EnumSet` et `EnumMap` pour les performances
- **Pattern matching** : intégration native avec le `switch` exhaustif
- **Machine à états** : chaque constante peut définir ses propres transitions

Disponibles depuis Java 5, les enums restent un pilier fondamental du langage et gagnent en puissance avec chaque nouvelle version de Java.

---

## Pour aller plus loin

- [Java Language Specification - Enum Types](https://docs.oracle.com/javase/specs/jls/se21/html/jls-8.html#jls-8.9)
- [Effective Java, Item 34: Use enums instead of int constants](https://www.oreilly.com/library/view/effective-java/9780134686097/)
- [EnumSet Javadoc](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/EnumSet.html)
- [EnumMap Javadoc](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/EnumMap.html)

## Voir aussi

- [Les Sealed classes en Java]({% post_url 2026-01-14-Sealed-classes-en-Java %})
- [Records en Java : simplifier vos DTOs]({% post_url 2026-01-10-Records-en-Java-simplifier-vos-DTOs %})
- [Introduction aux Streams en Java]({% post_url 2026-03-30-Introduction-aux-Streams-en-Java %})
- [Optional en Java : éviter les NullPointerException]({% post_url 2026-01-26-Optional-en-Java-eviter-les-NullPointerException %})
