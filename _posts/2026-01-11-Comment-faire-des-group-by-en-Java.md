---
layout: article
title: "Java : Comment faire des group by"
tags:
  - java
  - collections
  - streams
  - groupby
author: Pierre Chopinet
---

Regrouper des données par clé (faire un "group by") est une opération courante en programmation : calculer des statistiques par catégorie, compter des occurrences, agréger des montants. En Java, plusieurs approches existent selon vos besoins : boucles avec `Map`, Streams API avec `Collectors.groupingBy()`, ou bibliothèques tierces.
<!--more-->

Dans cet article :
- Groupement simple avec `Map` et boucles
- Groupement puissant avec Streams et `Collectors.groupingBy()`
- Agrégations avancées (count, sum, average, max, min)
- Groupement par plusieurs clés
- Utilisation avec les records (Java 16+)
- Pièges et bonnes pratiques

Pré-requis : Java 8 ou plus récent. Les exemples avec records nécessitent Java 16+.

---

## Jeu de données d'exemple

Nous utiliserons une liste de ventes :

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

---

## Approche classique : boucles et Map

Avant l'arrivée des Streams en Java 8, on utilisait des boucles et des `Map` pour regrouper des données. Cette approche reste valide et offre un contrôle total sur le processus.

### Grouper des objets par clé

Voici comment regrouper manuellement des ventes par ville :

```java
Map<String, List<Vente>> parVille = new HashMap<>();
for (Vente v : ventes) {
    parVille.computeIfAbsent(v.ville(), k -> new ArrayList<>()).add(v);
}

// Accès
parVille.get("Paris").forEach(System.out::println);
```

Explication :
- `computeIfAbsent` crée une nouvelle liste si la clé n'existe pas
- Chaque vente est ajoutée à la liste correspondante

### Compter les occurrences

Pour compter simplement le nombre d'éléments par groupe, utilisez `merge` :

```java
Map<String, Integer> compteParVille = new HashMap<>();
for (Vente v : ventes) {
    compteParVille.merge(v.ville(), 1, Integer::sum);
}

System.out.println(compteParVille); // {Paris=3, Lyon=2, Nantes=1}
```

`merge(key, 1, Integer::sum)` ajoute 1 si la clé existe, sinon initialise à 1.

### Calculer des totaux

Le même principe s'applique pour calculer des sommes ou d'autres agrégations :

```java
Map<String, Double> caParVille = new HashMap<>();
for (Vente v : ventes) {
    double montant = v.quantite() * v.prix();
    caParVille.merge(v.ville(), montant, Double::sum);
}

System.out.println(caParVille);
// {Paris=40.6, Lyon=31.0, Nantes=12.5}
```

---

## Streams API : Collectors.groupingBy()

Depuis Java 8, l'API Streams offre une approche déclarative et puissante pour regrouper des données avec `Collectors.groupingBy()`.

### Grouper des objets

La version avec Streams est beaucoup plus concise :

```java
Map<String, List<Vente>> parVille = ventes.stream()
    .collect(Collectors.groupingBy(Vente::ville));

parVille.forEach((ville, liste) ->
    System.out.println(ville + " : " + liste.size() + " ventes")
);
```

Avantages :
- Code concis et déclaratif
- Pas de gestion manuelle de la Map
- Composable avec d'autres opérations

### Compter les éléments par groupe

Pour compter les éléments de chaque groupe, utilisez le collector `counting()` :

```java
Map<String, Long> compteParVille = ventes.stream()
    .collect(Collectors.groupingBy(
        Vente::ville,
        Collectors.counting()
    ));

System.out.println(compteParVille);
// {Paris=3, Lyon=2, Nantes=1}
```

### Calculer une somme par groupe

Les collectors d'agrégation comme `summingDouble()` permettent de calculer des totaux directement :

```java
Map<String, Double> caParVille = ventes.stream()
    .collect(Collectors.groupingBy(
        Vente::ville,
        Collectors.summingDouble(v -> v.quantite() * v.prix())
    ));

System.out.println(caParVille);
// {Paris=40.6, Lyon=31.0, Nantes=12.5}
```

---

## Agrégations avancées

Au-delà des simples comptages et sommes, Java propose des collectors plus sophistiqués pour obtenir plusieurs statistiques en une seule passe.

### Plusieurs statistiques par groupe

Le collector `summarizingDouble()` calcule count, sum, average, min et max en une seule opération :

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

Sortie :
```
Paris : count=3, sum=40.60, avg=13.53, min=3.60, max=25.00
Lyon : count=2, sum=31.00, avg=15.50, min=6.00, max=25.00
Nantes : count=1, sum=12.50, avg=12.50, min=12.50, max=12.50
```

### Trouver le maximum ou minimum par groupe

Pour identifier l'élément avec la valeur maximale dans chaque groupe, utilisez `maxBy()` :

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

### Calculer une moyenne

Le collector `averagingDouble()` calcule la moyenne d'une valeur numérique pour chaque groupe :

```java
Map<String, Double> prixMoyenParVille = ventes.stream()
    .collect(Collectors.groupingBy(
        Vente::ville,
        Collectors.averagingDouble(Vente::prix)
    ));

System.out.println(prixMoyenParVille);
```

---

## Groupement par plusieurs clés

Parfois, on doit regrouper par plusieurs critères simultanément. Java offre deux approches : créer une clé composite ou imbriquer les groupements.

### Avec un record en clé composite

Créons un record pour la clé composite :

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

### Groupement imbriqué

Plutôt qu'une clé composite, on peut imbriquer les `groupingBy()` pour obtenir une structure hiérarchique :

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

Sortie :
```
Ville : Paris
  Livre : 1
  Stylo : 1
  Cahier : 1
Ville : Lyon
  Stylo : 1
  Livre : 1
Ville : Nantes
  Livre : 1
```

### Compter avec groupement imbriqué

Combinez les groupements imbriqués avec des collectors d'agrégation pour des statistiques détaillées :

```java
Map<String, Map<String, Long>> comptesParVilleEtProduit = ventes.stream()
    .collect(Collectors.groupingBy(
        Vente::ville,
        Collectors.groupingBy(
            Vente::produit,
            Collectors.counting()
        )
    ));
```

---

## Personnaliser le type de Map résultante

Par défaut, `groupingBy` retourne une `HashMap`. Si vous avez besoin d'un ordre spécifique ou d'autres propriétés, vous pouvez spécifier le type de Map.

### TreeMap pour un tri automatique

```java
Map<String, List<Vente>> parVilleTriee = ventes.stream()
    .collect(Collectors.groupingBy(
        Vente::ville,
        TreeMap::new,           // Map triée par clé
        Collectors.toList()
    ));
```

### LinkedHashMap pour conserver l'ordre d'insertion

```java
Map<String, Long> parVilleOrdonnee = ventes.stream()
    .collect(Collectors.groupingBy(
        Vente::ville,
        LinkedHashMap::new,
        Collectors.counting()
    ));
```

---

## Transformer les résultats après groupement

Au lieu de conserver les objets complets dans chaque groupe, on peut transformer ou extraire uniquement certaines valeurs avec `Collectors.mapping()`.

### Extraire seulement certaines valeurs

Pour ne garder qu'un champ spécifique de chaque objet groupé :

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
// {Paris=[Livre, Stylo, Cahier], Lyon=[Stylo, Livre], Nantes=[Livre]}
```

### Éliminer les doublons avec Set

Si certaines valeurs se répètent, utilisez `toSet()` pour ne conserver que les valeurs uniques :

```java
Map<String, Set<String>> produitsUniquesParVille = ventes.stream()
    .collect(Collectors.groupingBy(
        Vente::ville,
        Collectors.mapping(
            Vente::produit,
            Collectors.toSet()
        )
    ));
```

### Concaténer des chaînes

Pour obtenir une représentation textuelle des valeurs de chaque groupe, utilisez `joining()` :

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
// {Paris=Livre, Stylo, Cahier, Lyon=Stylo, Livre, Nantes=Livre}
```

---

## Filtrer avant ou après le groupement

Le filtrage peut s'effectuer à deux moments : avant de regrouper les éléments, ou après avoir créé les groupes pour ne garder que certains d'entre eux.

### Filtrer avant groupement

Pour ne regrouper que les éléments qui satisfont un critère :

```java
Map<String, List<Vente>> grossesVentesParVille = ventes.stream()
    .filter(v -> v.quantite() * v.prix() > 10)
    .collect(Collectors.groupingBy(Vente::ville));
```

### Filtrer les groupes après groupement

Pour ne conserver que les groupes qui répondent à une condition (par exemple, taille minimale) :

```java
Map<String, List<Vente>> villesAvecPlusieursVentes = ventes.stream()
    .collect(Collectors.groupingBy(Vente::ville))
    .entrySet()
    .stream()
    .filter(e -> e.getValue().size() > 1)
    .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));

System.out.println(villesAvecPlusieursVentes.keySet());
// [Paris, Lyon]
```

---

## Cas d'usage : construire un rapport d'agrégation

Combinons plusieurs techniques pour construire un rapport complet avec toutes les statistiques pertinentes par groupe.

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

Pour des cas plus avancés, regardez `Collectors.teeing()` (Java 12+) qui permet de combiner deux collectors en une seule passe.

---

## Avec les records (Java 16+)

Les records Java s'intègrent naturellement avec les opérations de groupement, offrant une syntaxe claire pour les modèles de données.

```java
public record Commande(String client, String statut, double montant) {}

List<Commande> commandes = List.of(
    new Commande("Alice", "LIVREE", 100.0),
    new Commande("Bob", "EN_COURS", 50.0),
    new Commande("Alice", "LIVREE", 75.0),
    new Commande("Charlie", "ANNULEE", 30.0)
);

Map<String, Double> montantParClient = commandes.stream()
    .filter(c -> "LIVREE".equals(c.statut()))
    .collect(Collectors.groupingBy(
        Commande::client,
        Collectors.summingDouble(Commande::montant)
    ));

System.out.println(montantParClient); // {Alice=175.0}
```

---

## Pièges et bonnes pratiques

### Pièges courants

Modifier la collection pendant le stream :
```java
// NE PAS FAIRE : ConcurrentModificationException
ventes.stream()
    .forEach(v -> ventes.remove(v)); // ERREUR
```

Grouper par valeur nulle :
```java
// Gérer les nulls avec un filtre ou une clé par défaut
ventes.stream()
    .filter(v -> v.ville() != null)
    .collect(Collectors.groupingBy(Vente::ville));
```

Oublier que `groupingBy` retourne une Map modifiable :
```java
// Rendre immuable si nécessaire
Map<String, List<Vente>> groupes = Collections.unmodifiableMap(
    ventes.stream().collect(Collectors.groupingBy(Vente::ville))
);
```

### À faire

- **Utilisez les records** pour les clés composites (immutables, `equals`/`hashCode` automatiques)
- **Préférez les Streams** pour la lisibilité et la composition
- **Choisissez le bon collector** selon vos besoins :
   - `counting()` pour compter
   - `summingDouble()` / `summingInt()` pour des totaux
   - `averagingDouble()` pour des moyennes
   - `summarizingDouble()` pour plusieurs stats en un coup
- **Nommez clairement vos variables** : `Map<String, List<Vente>> ventesParVille` plutôt que `Map<String, List<Vente>> map`
- **Documentez les groupements complexes** (multi-niveaux, agrégations multiples)
- **Testez les cas limites** : liste vide, valeurs nulles, un seul groupe

---

## Comparaison : boucles vs Streams

| Critère         | Boucles + Map          | Streams + groupingBy         |
|-----------------|------------------------|------------------------------|
| Lisibilité      | Moyenne                | Excellente                   |
| Performances    | Similaires             | Similaires (léger overhead)  |
| Parallélisation | Manuelle               | Facile (`.parallel()`)       |
| Composition     | Difficile              | Naturelle                    |
| Cas d'usage     | Contrôle fin, complexe | Transformations déclaratives |

Recommandation :
- Pour du code simple et déclaratif : **Streams**
- Pour des optimisations spécifiques ou logique complexe : **boucles classiques**

---

## Conclusion

Java offre plusieurs façons de faire des "group by", de l'approche classique avec boucles et `Map` jusqu'aux puissants Streams avec `Collectors.groupingBy()`.

**Récapitulatif :**
- **Boucles + Map** : contrôle total, verbeux
- **Streams + groupingBy** : concis, déclaratif, composable
- **Records** : clés composites élégantes
- **Collectors** : counting, summing, averaging, summarizing
- **Groupements imbriqués** : hiérarchies de Maps
- **Filtrage et transformation** : combinez avec `filter`, `map`, `mapping`

Les Streams et `groupingBy` sont aujourd'hui l'approche idiomatique en Java moderne.

---

## Pour aller plus loin

- [Collectors Javadoc](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/Collectors.html)
- [Stream API Guide](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/stream/package-summary.html)

## Voir aussi

- [Introduction aux Streams en Java]({% post_url 2026-03-30-Introduction-aux-Streams-en-Java %})
- [Les listes (List) en Java]({% post_url 2025-09-19-Framework-collections-java-list %})
- [Les maps (Map) en Java]({% post_url 2025-10-04-Framework-collections-java-map %})
- [Records en Java : simplifier vos DTOs]({% post_url 2026-01-10-Records-en-Java-simplifier-vos-DTOs %})
- [Pattern matching en Java moderne]({% post_url 2025-10-23-Pattern-matching-en-Java-moderne %})
- [Optional en Java : éviter les NullPointerException]({% post_url 2026-01-26-Optional-en-Java-eviter-les-NullPointerException %})
