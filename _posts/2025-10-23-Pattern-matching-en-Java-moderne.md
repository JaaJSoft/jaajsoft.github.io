---
layout: article
title: "Pattern matching en Java moderne"
author: Pierre Chopinet
tags:
  - java
  - pattern-matching
  - tutoriel
---

Le pattern matching a profondément simplifié l’écriture de code orienté données en Java. Finalisé dans Java 21 pour `switch` et les « record patterns », il s’impose désormais comme un outil idiomatique dans le Java moderne.
<!--more-->

Objectifs de l’article :
- Comprendre le pattern matching pour `instanceof` et `switch`
- Découvrir les record patterns et leur combinaison avec `switch`
- Utiliser les `when` et écrire des `switch` exhaustifs
- Tirer parti des classes `sealed` pour des hiérarchies sûres
- Connaître les limites, pièges et bonnes pratiques

Pré‑requis : Java 17+ conseillé (LTS). Les fonctionnalités présentées comme « finalisées » le sont dès Java 21.

---

## 1) Rappel : pattern matching pour instanceof

Avant Java 16/17, on écrivait :

```java
if (obj instanceof String) {
    String s = (String) obj; // cast explicite
    System.out.println(s.toUpperCase());
}
```

Avec le pattern matching pour `instanceof` :

```java
if (obj instanceof String s) {
    System.out.println(s.toUpperCase()); // s est déjà typé
}
```

- Le binding (`String s`) est introduit dans la condition.
- Le scope de `s` est limité au bloc où le test est vrai.

---

## 2) Pattern matching pour switch (Java 21)

Le `switch` accepte des patterns de type et valeur (`when`).

Exemple simple :

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

Points clés :
- `case null` est possible et utile pour éliminer les NPE.
- Les patterns sont testés dans l’ordre, la première correspondance gagne.
- Les gardes `when` affinent un pattern par une condition booléenne.
- Le compilateur vérifie l’exhaustivité (selon les types, notamment avec `sealed`).

---

## 3) Record patterns (Java 21)

Les records permettent de déstructurer un objet par ses composants, directement dans le `case`.

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

- `Point(int x, int y)` lie `x`/`y` sans écrire d’accesseurs explicitement.
- Les gardes permettent d’exprimer la logique métier localement.

### Nesting (imbriquer des patterns)

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

---

## 4) sealed + patterns : des hiérarchies fermées et exhaustives

Les classes scellées permettent de contrôler les sous‑types et aident le compilateur à vérifier l’exhaustivité des `switch`.

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
    }; // exhaustif : pas de default nécessaire
}
```

Ici, l’absence de `default` est possible, car la hiérarchie est connue (grâce à `sealed`). En cas d’ajout d’un nouveau sous‑type autorisé, le compilateur signalera les `switch` non à jour.

---

## 5) Dominance, ordre des case et variable shadowing

- Placez les `case` plus spécifiques avant les plus génériques, sinon les spécifiques deviennent inatteignables.

```java
static String f(Object o) {
    return switch (o) {
        case String s when s.length() > 10 -> "long string";
        case String s -> "string";
        case Object x -> "object";
    };
}
```

- Le nom des variables de binding doit être unique par alternative ; évitez les collisions avec des variables existantes dans la portée.

---

## 6) Null et switch

- `case null` est supporté et recommandé si `o` peut être `null`.
- Sans `case null` ni `default`, un `switch` sur une référence `null` lancerait un `NullPointerException`.

---

## 7) Bonnes pratiques

- Préférez les `switch` expression (`switch (...) { ... }`) pour des retours clairs et immutables.
- Limitez la logique dans les gardes `when` : gardes simples, logique métier ailleurs.
- Combinez `sealed` + records + patterns pour coder des « sum types » lisibles.
- Conservez l’exhaustivité : évitez `default` quand une hiérarchie scellée la rend vérifiable.
- Gardez les `case` courts ; extraire en méthodes si nécessaire.

---

## 8) Pièges fréquents

- Un `case` générique (`Object o`) placé trop tôt capture tout et rend les suivants inaccessibles.
- Gardes avec effets de bord : évitez d’appeler des méthodes non idempotentes dans `when`.
- Ne confondez pas record patterns et déconstruction arbitraire : seuls les records (ou patterns définis) sont déstructurables de cette façon.
- Attention aux `switch` non exhaustifs sur des hiérarchies non `sealed` : gardez un `default` sensé.

---

## FAQ

- Peut‑on utiliser les patterns avec des types primitifs ?
  - Les patterns de type s’appliquent aux références ; pour les primitifs, on continue d’utiliser les `case` littéraux (`case 1, 2, 3 -> ...`).
- Est‑ce disponible en Java 17 ?
  - `instanceof` avec binding fonctionne. Les `switch`/record patterns finalisés arrivent en Java 21. Sur Java 17, certaines fonctionnalités n’existent pas encore.

---

## Conclusion

Le pattern matching apporte des `switch` plus lisibles, moins de casts et des logiques déclaratives puissantes, surtout combiné avec `records` et `sealed`. Finalisé en Java 21, c’est un incontournable du Java moderne.

---

## Pour aller plus loin

- JEP 441 : Pattern Matching for switch (Final, JDK 21)
- JEP 440 : Record Patterns (Final, JDK 21)
- JEP 409 : Sealed Classes (Final, JDK 17)
- Javadoc : `switch` expressions et statements (Java 21+)

## Voir aussi

- [Introduction aux collections Java]({% post_url 2020-11-12-Framework-collections-java-intro %})
- [Les listes (List) en Java]({% post_url 2025-09-19-Framework-collections-java-list %})
- [Les ensembles (Set) en Java]({% post_url 2025-09-25-Framework-collections-java-set %})
- [Les files (Queue) et Deques en Java]({% post_url 2025-09-26-Framework-collections-java-queue %})
- [Les maps (Map) en Java]({% post_url 2025-10-04-Framework-collections-java-map %})
