---
layout: article
title: "Java : Comment faire des group by"
description: "Faire des group by en Java avec des boucles et une Map ou avec Collectors.groupingBy : agrégations, clés multiples, filtrage et rapports avec les records."
tags:
  - java
  - collections
  - streams
  - groupby
author: Pierre Chopinet
---

Compter des ventes par ville, additionner un chiffre d'affaires par produit, trouver la plus grosse commande de chaque client : ce sont des "group by", comme en SQL. En Java, on peut les écrire avec une boucle et une `Map`, ou avec les Streams et `Collectors.groupingBy`, qui couvre la plupart des agrégations en une seule expression.
<!--more-->

Dans cet article :
- Le jeu de données
- Regrouper avec une boucle et une Map
- Regrouper avec `Collectors.groupingBy`
- Plusieurs statistiques par groupe
- Grouper sur plusieurs clés
- Choisir le type de Map
- Transformer les éléments de chaque groupe
- Filtrer avant ou après le regroupement
- Construire un rapport par ville
- Les clés null

Pré-requis : connaître les bases des [Streams]({% post_url 2026-03-30-Introduction-aux-Streams-en-Java %}). Les exemples utilisent des records et `Stream.toList()`, et demandent donc Java 16 ou plus récent. Ils ont été testés avec Java 21.

## Le jeu de données

Tous les exemples utilisent la même liste de ventes, celle de la [version Python de cet article]({% post_url 2025-10-08-Comment-faire-des-group-by-en-python %}) :

```java
public record Vente(String ville, String produit, int quantite, double prix) {}

List<Vente> ventes = List.of(
    new Vente("Paris", "Livre", 2, 12.5),
    new Vente("Lyon", "Stylo", 5, 1.2),
    new Vente("Paris", "Stylo", 3, 1.2),
    new Vente("Nantes", "Livre", 1, 12.5),
    new Vente("Paris", "Cahier", 4, 3.0),
    new Vente("Lyon", "Livre", 2, 12.5)
);
```

## Regrouper avec une boucle et une Map

Avant les Streams, on parcourait les ventes en rangeant chacune dans la liste de sa ville. `computeIfAbsent` crée la liste la première fois qu'une ville est rencontrée, et la retourne dans tous les cas :

```java
Map<String, List<Vente>> parVille = new HashMap<>();
for (Vente v : ventes) {
    parVille.computeIfAbsent(v.ville(), k -> new ArrayList<>()).add(v);
}

// Accès
parVille.get("Paris").forEach(System.out::println);
```

On récupère les trois ventes de Paris :

```
Vente[ville=Paris, produit=Livre, quantite=2, prix=12.5]
Vente[ville=Paris, produit=Stylo, quantite=3, prix=1.2]
Vente[ville=Paris, produit=Cahier, quantite=4, prix=3.0]
```

Pour compter les ventes de chaque ville, pas besoin de garder les listes : `merge` ajoute 1 au compteur de la ville, ou l'initialise à 1 si la ville n'est pas encore dans la map.

```java
Map<String, Integer> compteParVille = new HashMap<>();
for (Vente v : ventes) {
    compteParVille.merge(v.ville(), 1, Integer::sum);
}

System.out.println(compteParVille); // {Nantes=1, Lyon=2, Paris=3}
```

L'ordre d'itération d'une `HashMap` n'est pas garanti : les villes pourraient sortir dans un autre ordre. Ne basez aucune logique dessus.

Le même principe permet de calculer le chiffre d'affaires de chaque ville :

```java
Map<String, Double> caParVille = new HashMap<>();
for (Vente v : ventes) {
    double montant = v.quantite() * v.prix();
    caParVille.merge(v.ville(), montant, Double::sum);
}

System.out.println(caParVille);
// {Nantes=12.5, Lyon=31.0, Paris=40.6}
```

Ces deux méthodes de `Map` sont détaillées dans l'article sur [les maps en Java]({% post_url 2025-10-04-Framework-collections-java-map %}).

## Regrouper avec `Collectors.groupingBy`

Avec les Streams, `Collectors.groupingBy` fait le même travail. On lui donne la fonction qui calcule la clé de chaque élément, et il retourne une `Map` dont chaque valeur est la liste des éléments qui ont cette clé :

```java
Map<String, List<Vente>> parVille = ventes.stream()
    .collect(Collectors.groupingBy(Vente::ville));

parVille.forEach((ville, liste) ->
    System.out.println(ville + " : " + liste.size() + " ventes")
);
```

```
Nantes : 1 ventes
Lyon : 2 ventes
Paris : 3 ventes
```

Le deuxième paramètre de `groupingBy`, facultatif, est un autre collector, qui dit quoi faire des éléments de chaque groupe au lieu de les mettre dans une liste. `Collectors.counting()` les compte :

```java
Map<String, Long> compteParVille = ventes.stream()
    .collect(Collectors.groupingBy(
        Vente::ville,
        Collectors.counting()
    ));

System.out.println(compteParVille);
// {Nantes=1, Lyon=2, Paris=3}
```

Et `summingDouble` (ou `summingInt` et `summingLong`) additionne une valeur calculée pour chaque élément :

```java
Map<String, Double> caParVille = ventes.stream()
    .collect(Collectors.groupingBy(
        Vente::ville,
        Collectors.summingDouble(v -> v.quantite() * v.prix())
    ));

System.out.println(caParVille);
// {Nantes=12.5, Lyon=31.0, Paris=40.6}
```

On obtient les mêmes résultats qu'avec les boucles, sans gérer la map soi-même. Notez seulement que `counting()` compte avec des `Long`, là où la boucle utilisait des `Integer`. Dès qu'on combine plusieurs agrégations, comme dans la suite de l'article, la version Stream devient nettement plus courte. La boucle reste plus simple à lire quand le traitement de chaque élément demande plusieurs étapes.

Sur un Stream parallèle, `groupingBy` doit fusionner les maps construites par chaque thread, ce qui peut coûter cher. Sa documentation suggère dans ce cas `groupingByConcurrent`, qui remplit une `ConcurrentMap`, si l'ordre des éléments dans chaque groupe n'a pas d'importance.

## Plusieurs statistiques par groupe

`summarizingDouble` calcule en une seule passe le nombre d'éléments, la somme, la moyenne, le minimum et le maximum de chaque groupe, dans un objet `DoubleSummaryStatistics` :

```java
Map<String, DoubleSummaryStatistics> statsParVille = ventes.stream()
    .collect(Collectors.groupingBy(
        Vente::ville,
        Collectors.summarizingDouble(v -> v.quantite() * v.prix())
    ));

statsParVille.forEach((ville, stats) -> {
    System.out.printf("%s : count=%d, sum=%.2f, avg=%.2f, min=%.2f, max=%.2f%n",
        ville,
        stats.getCount(),
        stats.getSum(),
        stats.getAverage(),
        stats.getMin(),
        stats.getMax()
    );
});
```

Ce qui donne :

```
Nantes : count=1, sum=12.50, avg=12.50, min=12.50, max=12.50
Lyon : count=2, sum=31.00, avg=15.50, min=6.00, max=25.00
Paris : count=3, sum=40.60, avg=13.53, min=3.60, max=25.00
```

Pour récupérer l'élément qui a la plus grande valeur dans chaque groupe, et pas seulement cette valeur, on utilise `maxBy` (ou `minBy`) avec un comparateur :

```java
Map<String, Optional<Vente>> ventesMaxParVille = ventes.stream()
    .collect(Collectors.groupingBy(
        Vente::ville,
        Collectors.maxBy(Comparator.comparingDouble(v -> v.quantite() * v.prix()))
    ));

ventesMaxParVille.forEach((ville, opt) ->
    opt.ifPresent(v -> System.out.println(ville + " : " + v))
);
```

```
Nantes : Vente[ville=Nantes, produit=Livre, quantite=1, prix=12.5]
Lyon : Vente[ville=Lyon, produit=Livre, quantite=2, prix=12.5]
Paris : Vente[ville=Paris, produit=Livre, quantite=2, prix=12.5]
```

`maxBy` retourne un `Optional`, qui serait vide s'il n'avait reçu aucun élément. Dans un `groupingBy`, chaque groupe contient au moins un élément, l'`Optional` est donc toujours rempli. Pour obtenir directement la vente, on peut envelopper le collector : `Collectors.collectingAndThen(Collectors.maxBy(...), Optional::get)`.

Enfin, `averagingDouble` calcule une moyenne. Ici, la moyenne des prix unitaires des ventes de chaque ville, sans tenir compte des quantités :

```java
Map<String, Double> prixMoyenParVille = ventes.stream()
    .collect(Collectors.groupingBy(
        Vente::ville,
        Collectors.averagingDouble(Vente::prix)
    ));

System.out.println(prixMoyenParVille);
// {Nantes=12.5, Lyon=6.85, Paris=5.566666666666666}
```

## Grouper sur plusieurs clés

Pour regrouper par ville et par produit, il y a deux possibilités. La première est d'utiliser une clé composite. Un record s'y prête bien, puisque ses méthodes `equals` et `hashCode` sont générées à partir de ses composants (voir [Records en Java : simplifier vos DTOs]({% post_url 2026-01-10-Records-en-Java-simplifier-vos-DTOs %})) :

```java
public record VilleProduit(String ville, String produit) {}

Map<VilleProduit, List<Vente>> parVilleEtProduit = ventes.stream()
    .collect(Collectors.groupingBy(
        v -> new VilleProduit(v.ville(), v.produit())
    ));

parVilleEtProduit.forEach((cle, liste) ->
    System.out.printf("%s - %s : %d vente(s)%n",
        cle.ville(), cle.produit(), liste.size())
);
```

```
Lyon - Stylo : 1 vente(s)
Paris - Livre : 1 vente(s)
Nantes - Livre : 1 vente(s)
Lyon - Livre : 1 vente(s)
Paris - Stylo : 1 vente(s)
Paris - Cahier : 1 vente(s)
```

La seconde est d'imbriquer deux `groupingBy` : le collector passé en deuxième paramètre est lui-même un regroupement, et on obtient une map de maps.

```java
Map<String, Map<String, List<Vente>>> parVillePuisProduit = ventes.stream()
    .collect(Collectors.groupingBy(
        Vente::ville,
        Collectors.groupingBy(Vente::produit)
    ));

// Parcours
parVillePuisProduit.forEach((ville, parProduit) -> {
    System.out.println("Ville : " + ville);
    parProduit.forEach((produit, liste) ->
        System.out.println("  " + produit + " : " + liste.size())
    );
});
```

```
Ville : Nantes
  Livre : 1
Ville : Lyon
  Stylo : 1
  Livre : 1
Ville : Paris
  Cahier : 1
  Stylo : 1
  Livre : 1
```

Et comme pour un regroupement simple, on peut remplacer les listes par une agrégation en la passant au `groupingBy` intérieur :

```java
Map<String, Map<String, Long>> comptesParVilleEtProduit = ventes.stream()
    .collect(Collectors.groupingBy(
        Vente::ville,
        Collectors.groupingBy(
            Vente::produit,
            Collectors.counting()
        )
    ));

System.out.println(comptesParVilleEtProduit);
// {Nantes={Livre=1}, Lyon={Stylo=1, Livre=1}, Paris={Cahier=1, Stylo=1, Livre=1}}
```

## Choisir le type de Map

La documentation de `groupingBy` ne garantit ni le type ni la mutabilité de la map retournée (en pratique, c'est une `HashMap`). Pour choisir, on passe une fabrique en deuxième paramètre, et le collector des valeurs devient le troisième. Avec une `TreeMap`, les villes sont triées :

```java
Map<String, List<Vente>> parVilleTriee = ventes.stream()
    .collect(Collectors.groupingBy(
        Vente::ville,
        TreeMap::new,           // Map triée par clé
        Collectors.toList()
    ));

System.out.println(parVilleTriee.keySet()); // [Lyon, Nantes, Paris]
```

Avec une `LinkedHashMap`, elles restent dans l'ordre de leur première apparition dans la liste :

```java
Map<String, Long> parVilleOrdonnee = ventes.stream()
    .collect(Collectors.groupingBy(
        Vente::ville,
        LinkedHashMap::new,
        Collectors.counting()
    ));

System.out.println(parVilleOrdonnee); // {Paris=3, Lyon=2, Nantes=1}
```

Pour obtenir une map non modifiable, on peut envelopper le résultat avec `Collections.unmodifiableMap`. Seule la map est protégée : les listes qu'elle contient restent modifiables.

```java
Map<String, List<Vente>> groupes = Collections.unmodifiableMap(
    ventes.stream().collect(Collectors.groupingBy(Vente::ville))
);
```

## Transformer les éléments de chaque groupe

On n'a pas toujours besoin des objets complets dans chaque groupe. `Collectors.mapping` applique une fonction à chaque élément avant de le passer au collector suivant, par exemple pour ne garder que le nom du produit :

```java
Map<String, List<String>> produitsParVille = ventes.stream()
    .collect(Collectors.groupingBy(
        Vente::ville,
        Collectors.mapping(
            Vente::produit,
            Collectors.toList()
        )
    ));

System.out.println(produitsParVille);
// {Nantes=[Livre], Lyon=[Stylo, Livre], Paris=[Livre, Stylo, Cahier]}
```

Avec `Collectors.toSet()` à la place de `Collectors.toList()`, chaque produit n'apparaît qu'une fois par ville, dans un ensemble dont l'ordre n'est pas garanti. Et avec `joining`, on obtient directement une chaîne :

```java
Map<String, String> produitsJointsParVille = ventes.stream()
    .collect(Collectors.groupingBy(
        Vente::ville,
        Collectors.mapping(
            Vente::produit,
            Collectors.joining(", ")
        )
    ));

System.out.println(produitsJointsParVille);
// {Nantes=Livre, Lyon=Stylo, Livre, Paris=Livre, Stylo, Cahier}
```

## Filtrer avant ou après le regroupement

Pour ne regrouper qu'une partie des éléments, on filtre le Stream avant le `groupingBy`. Ici, on ne garde que les ventes de plus de 10 euros :

```java
Map<String, List<Vente>> grossesVentesParVille = ventes.stream()
    .filter(v -> v.quantite() * v.prix() > 10)
    .collect(Collectors.groupingBy(Vente::ville));
```

Une ville dont aucune vente ne passe le filtre disparaît alors du résultat. Pour la garder avec un groupe vide, il faut filtrer à l'intérieur du regroupement, avec `Collectors.filtering` (Java 9+). Avec un seuil de 20 euros :

```java
Map<String, Long> grossesVentes = ventes.stream()
    .collect(Collectors.groupingBy(
        Vente::ville,
        Collectors.filtering(v -> v.quantite() * v.prix() > 20, Collectors.counting())
    ));

System.out.println(grossesVentes); // {Nantes=0, Lyon=1, Paris=1}
```

Avec le même seuil dans un `filter` placé avant le regroupement, Nantes, dont la seule vente fait 12,50 euros, n'apparaîtrait pas du tout : on obtiendrait `{Lyon=1, Paris=1}`.

Pour filtrer les groupes eux-mêmes, par exemple ne garder que les villes qui ont plusieurs ventes, il faut construire la map, puis repartir de ses entrées :

```java
Map<String, List<Vente>> villesAvecPlusieursVentes = ventes.stream()
    .collect(Collectors.groupingBy(Vente::ville))
    .entrySet()
    .stream()
    .filter(e -> e.getValue().size() > 1)
    .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));

System.out.println(villesAvecPlusieursVentes.keySet());
// [Lyon, Paris]
```

## Construire un rapport par ville

Quand on veut plusieurs informations par groupe, on peut regrouper les ventes, puis construire un objet par groupe à partir de sa liste :

```java
public record RapportVille(
    String ville,
    long nombreVentes,
    double chiffreAffaires,
    double montantMoyen,
    Set<String> produits
) {}

Map<String, List<Vente>> groupes = ventes.stream()
    .collect(Collectors.groupingBy(Vente::ville));

List<RapportVille> rapports = groupes.entrySet().stream()
    .map(e -> {
        String ville = e.getKey();
        List<Vente> liste = e.getValue();
        long count = liste.size();
        double ca = liste.stream()
            .mapToDouble(v -> v.quantite() * v.prix())
            .sum();
        Set<String> prods = liste.stream()
            .map(Vente::produit)
            .collect(Collectors.toSet());
        return new RapportVille(ville, count, ca, ca / count, prods);
    })
    .toList();

rapports.forEach(System.out::println);
```

```
RapportVille[ville=Nantes, nombreVentes=1, chiffreAffaires=12.5, montantMoyen=12.5, produits=[Livre]]
RapportVille[ville=Lyon, nombreVentes=2, chiffreAffaires=31.0, montantMoyen=15.5, produits=[Stylo, Livre]]
RapportVille[ville=Paris, nombreVentes=3, chiffreAffaires=40.6, montantMoyen=13.533333333333333, produits=[Cahier, Stylo, Livre]]
```

Cette version parcourt chaque groupe plusieurs fois, ce qui ne gêne pas sur de petites listes. `Collectors.teeing` (Java 12+) permet de calculer deux agrégations en une seule passe : chaque élément est envoyé à deux collectors, dont les résultats sont ensuite combinés. Par exemple, pour un bilan avec le nombre de ventes et le chiffre d'affaires de chaque ville :

```java
record Bilan(long ventes, double chiffreAffaires) {}

Map<String, Bilan> bilanParVille = ventes.stream()
    .collect(Collectors.groupingBy(
        Vente::ville,
        Collectors.teeing(
            Collectors.counting(),
            Collectors.summingDouble(v -> v.quantite() * v.prix()),
            Bilan::new
        )
    ));

System.out.println(bilanParVille);
```

```
{Nantes=Bilan[ventes=1, chiffreAffaires=12.5], Lyon=Bilan[ventes=2, chiffreAffaires=31.0], Paris=Bilan[ventes=3, chiffreAffaires=40.6]}
```

## Les clés null

`groupingBy` refuse les clés `null`. Si la liste contenait une vente sans ville, `new Vente(null, "Stylo", 2, 1.2)` par exemple, le regroupement par ville s'arrêterait sur une `NullPointerException: element cannot be mapped to a null key`. La boucle avec `merge` n'aurait pas ce problème, puisqu'une `HashMap` accepte une clé `null`.

On peut écarter ces éléments avant le regroupement, ou les ranger sous une clé par défaut avec `Objects.requireNonNullElse` (Java 9+) :

```java
// Ignorer les ventes sans ville
ventes.stream()
    .filter(v -> v.ville() != null)
    .collect(Collectors.groupingBy(Vente::ville));

// Ou les compter sous une clé par défaut
ventes.stream()
    .collect(Collectors.groupingBy(
        v -> Objects.requireNonNullElse(v.ville(), "Inconnue"),
        Collectors.counting()
    ));
// Avec la vente sans ville : {Inconnue=1, Nantes=1, Lyon=2, Paris=3}
```

## Voir aussi

- [Introduction aux Streams en Java]({% post_url 2026-03-30-Introduction-aux-Streams-en-Java %})
- [Les maps (Map) en Java]({% post_url 2025-10-04-Framework-collections-java-map %})
- [Records en Java : simplifier vos DTOs]({% post_url 2026-01-10-Records-en-Java-simplifier-vos-DTOs %})
- [Optional en Java : éviter les NullPointerException]({% post_url 2026-01-26-Optional-en-Java-eviter-les-NullPointerException %})
- [Python : Comment faire des group by]({% post_url 2025-10-08-Comment-faire-des-group-by-en-python %})
- [Javadoc de Collectors (Java 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/Collectors.html)
- [Javadoc du package java.util.stream (Java 21)](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/package-summary.html)
