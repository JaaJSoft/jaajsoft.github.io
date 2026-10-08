---
layout: article
title: "Pattern matching en Java moderne"
description: "Le pattern matching en Java : instanceof, switch et record patterns de Java 21, null, ordre des case, switch exhaustifs, classes scellées, types primitifs."
tags:
  - java
  - pattern-matching
author: Pierre Chopinet
---

Tester le type d'un objet, le caster, puis lire ses champs : en Java, cette suite d'opérations a longtemps demandé beaucoup de code. Avec le pattern matching, complété en Java 21, tout ça se fait directement dans un `instanceof` ou dans un `switch`, y compris avec les records et les classes scellées.
<!--more-->

Dans cet article :
- Pattern matching pour instanceof
- Pattern matching pour switch
- Le cas de null
- L'ordre des case
- Les record patterns
- Switch exhaustifs avec les classes scellées
- Les types primitifs

Pré-requis : Java 21 ou plus récent pour le `switch` et les record patterns (avant Java 21, ils n'existaient qu'en preview). Le pattern matching pour `instanceof` fonctionne dès Java 16.

## Pattern matching pour instanceof

Avant Java 16, utiliser un objet après avoir testé son type demandait un cast explicite :

```java
if (obj instanceof String) {
    String s = (String) obj; // cast explicite
    System.out.println(s.toUpperCase());
}
```

Depuis Java 16, `instanceof` accepte un *pattern* : la variable est déclarée directement dans le test.

```java
if (obj instanceof String s) {
    System.out.println(s.toUpperCase()); // s est déjà typé
}
```

La variable `s` n'existe que là où le compilateur sait que le test est vrai. On peut donc s'en servir dans la suite d'une condition avec `&&` :

```java
if (obj instanceof String s && s.length() > 3) {
    System.out.println("Chaîne longue : " + s);
}
```

Ou après un test inversé qui sort de la méthode :

```java
if (!(obj instanceof String s)) {
    return -1;
}
return s.length(); // s est utilisable ici
```

Attention, cette variable ne peut pas porter le nom d'une variable locale déjà visible : si un `String s` est déclaré plus haut dans la méthode, le compilateur refuse le code avec `variable s is already defined`.

## Pattern matching pour switch

Java 21 étend le principe au `switch` : chaque `case` peut tester un type, et le mot-clé `when` ajoute une condition (on parle de garde).

```java
static String render(Object o) {
    return switch (o) {
        case null -> "<null>";                    // null est géré explicitement
        case Integer i -> "int=" + i;
        case Long l -> "long=" + l;
        case String s when s.isBlank() -> "<empty string>"; // garde
        case String s -> "str='" + s + "'";
        default -> "autre=" + o.getClass().getSimpleName();
    };
}
```

Testons avec quelques valeurs :

```java
System.out.println(render(null));
System.out.println(render(42));
System.out.println(render(42L));
System.out.println(render("   "));
System.out.println(render("Java"));
System.out.println(render(3.14));
```

Ce qui donne :

```
<null>
int=42
long=42
<empty string>
str='Java'
autre=Double
```

Les `case` sont testés dans l'ordre et le premier qui correspond l'emporte : la chaîne `"   "` passe par la garde `when s.isBlank()` et n'atteint jamais le `case String s` suivant. La variable `s` peut être déclarée dans plusieurs `case`, car sa portée se limite à sa branche.

Le `default` n'est pas là pour décorer. Un `switch` qui utilise des patterns doit être exhaustif, même quand il est utilisé comme instruction et pas comme expression. Sur un `Object`, sans `default`, la compilation échoue avec `the switch expression does not cover all possible input values` (ou `the switch statement ...` pour une instruction).

Les conditions `when` gagnent à rester courtes et sans effet de bord. Si une garde commence à déborder sur plusieurs lignes, une méthode avec un nom explicite sera plus lisible.

## Le cas de null

Un `switch` sur une référence `null` lance une `NullPointerException`, et c'est toujours le cas en Java 21 quand il n'y a pas de `case null`. Attention, un `default` ne suffit pas à l'éviter :

```java
static String withDefault(Object o) {
    return switch (o) {
        case String s -> "string";
        default -> "autre";
    };
}

withDefault(null); // NullPointerException
```

Pour traiter `null` comme n'importe quelle autre valeur inattendue, on regroupe `case null` et `default` :

```java
static String nullDefault(Object o) {
    return switch (o) {
        case String s -> "string";
        case null, default -> "autre ou null";
    };
}
```

Cette fois, `nullDefault(null)` renvoie `"autre ou null"`.

## L'ordre des case

Puisque le premier `case` qui correspond gagne, les plus spécifiques doivent être placés en premier :

```java
static String f(Object o) {
    return switch (o) {
        case String s when s.length() > 10 -> "long string";
        case String s -> "string";
        case Object x -> "object";
    };
}
```

Le `case Object x` accepte n'importe quel objet : il joue le rôle du `default` et rend le `switch` exhaustif.

Si on inverse les deux premiers `case`, la version avec `when` ne peut plus jamais être atteinte. Le compilateur s'en rend compte et refuse le code avec l'erreur `this case label is dominated by a preceding case label` : on dit que le premier `case` *domine* le second. Un `case Object x` placé en tête dominerait de la même façon tous ceux qui le suivent.

## Les record patterns

Un `case` (ou un `instanceof`) peut aussi déstructurer un record. Le pattern `Point(int x, int y)` vérifie le type et récupère directement les composants, sans avoir à appeler les accesseurs :

```java
record Point(int x, int y) {}

static String quadrant(Object o) {
    return switch (o) {
        case Point(int x, int y) when x == 0 && y == 0 -> "origin";
        case Point(int x, int y) when x >= 0 && y >= 0 -> "Q1";
        case Point(int x, int y) when x < 0 && y >= 0 -> "Q2";
        case Point(int x, int y) when x < 0 && y < 0 -> "Q3";
        case Point(int x, int y) when x >= 0 && y < 0 -> "Q4";
        default -> "n/a";
    };
}
```

```java
System.out.println(quadrant(new Point(0, 0)));
System.out.println(quadrant(new Point(3, 4)));
System.out.println(quadrant(new Point(-2, 5)));
System.out.println(quadrant("pas un point"));
```

On obtient :

```
origin
Q1
Q2
n/a
```

Le type de chaque composant peut être remplacé par `var` : `case Point(var x, var y)` fonctionne tout aussi bien.

Seuls les records se déstructurent de cette façon. Sur une classe classique, le compilateur répond `deconstruction patterns can only be applied to records`.

### Patterns imbriqués

Les patterns s'imbriquent, ce qui permet de descendre dans une structure en une seule ligne :

```java
record Line(Point start, Point end) {}

static int manhattan(Object o) {
    return switch (o) {
        case Line(Point(int x1, int y1), Point(int x2, int y2)) ->
            Math.abs(x1 - x2) + Math.abs(y1 - y2);
        default -> 0;
    };
}
```

`manhattan(new Line(new Point(0, 0), new Point(3, 4)))` renvoie `7`. Attention, un pattern imbriqué ne correspond jamais à un composant `null` : `new Line(null, new Point(3, 4))` ne passe pas par le premier `case` et finit dans le `default`, qui renvoie `0`.

### Ignorer un composant avec `_`

Quand un composant ne sert à rien, on peut le remplacer par `_`. Ces patterns et variables anonymes sont définitifs depuis Java 22 (JEP 456). En Java 21, ils étaient encore en preview (JEP 443) et demandaient l'option `--enable-preview`.

```java
static String axe(Object o) {
    return switch (o) {
        case Point(var x, _) when x == 0 -> "sur l'axe vertical";
        case Point _ -> "un point";
        default -> "autre chose";
    };
}
```

Ici, `axe(new Point(0, 5))` renvoie `"sur l'axe vertical"` et `axe(new Point(1, 5))` renvoie `"un point"`.

## Switch exhaustifs avec les classes scellées

Sur un `Object`, le compilateur ne peut pas connaître tous les types possibles, d'où le `default` des exemples précédents. Avec une interface scellée, il les connaît : la liste des sous-types autorisés est fermée (le sujet est détaillé dans l'article sur [les classes scellées]({% post_url 2026-01-14-Sealed-classes-en-Java %})).

```java
sealed interface Shape permits Circle, Rectangle, Triangle {}
record Circle(double r) implements Shape {}
record Rectangle(double w, double h) implements Shape {}
record Triangle(double a, double b, double c) implements Shape {}

static double area(Shape s) {
    return switch (s) {
        case Circle(double r) -> Math.PI * r * r;
        case Rectangle(double w, double h) -> w * h;
        case Triangle(double a, double b, double c) -> heron(a, b, c);
    }; // exhaustif : pas de default nécessaire
}

// Formule de Héron : aire d'un triangle à partir de ses trois côtés
static double heron(double a, double b, double c) {
    double p = (a + b + c) / 2;
    return Math.sqrt(p * (p - a) * (p - b) * (p - c));
}
```

`area(new Circle(1))` renvoie `3.141592653589793` et `area(new Triangle(3, 4, 5))` renvoie `6.0`.

L'intérêt de se passer du `default` apparaît le jour où on ajoute un sous-type. Si un `record Square(double side)` rejoint la liste `permits`, ce `switch` ne compile plus (`the switch expression does not cover all possible input values`). Le compilateur signale ainsi chaque `switch` à mettre à jour, là où un `default` aurait avalé le nouveau cas sans rien dire.

## Les types primitifs

En Java 21, les patterns de type ne s'appliquent qu'aux types référence. Un `case int i` dans un `switch` sur un `int` est refusé (`unexpected type`). On continue donc d'utiliser des constantes (`case 1, 2, 3 -> ...`). Les record patterns, eux, déstructurent sans problème des composants primitifs, comme `Point(int x, int y)` plus haut.

Les patterns sur les types primitifs (`case int i when i < 0`, `instanceof int`) sont en preview depuis Java 23 (JEP 455), et le sont encore en Java 25 : il faut `--enable-preview` pour s'en servir.

## Voir aussi

- [Records en Java : simplifier vos DTOs]({% post_url 2026-01-10-Records-en-Java-simplifier-vos-DTOs %})
- [Les Sealed classes en Java]({% post_url 2026-01-14-Sealed-classes-en-Java %})
- [Python : Le pattern matching avec match et case]({% post_url 2026-05-18-Python-pattern-matching-avec-match-et-case %})
- [JEP 441 : Pattern Matching for switch](https://openjdk.org/jeps/441)
- [JEP 440 : Record Patterns](https://openjdk.org/jeps/440)
- [JEP 456 : variables et patterns anonymes](https://openjdk.org/jeps/456)
- [Guide Oracle sur le pattern matching](https://docs.oracle.com/en/java/javase/21/language/pattern-matching.html)
