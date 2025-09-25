---
layout: article
title: Introduction aux collections Java
tags:
    - java
    - collections
author: Julien Chevron

---

Dans cet article, nous allons nous attarder sur une des fonctionnalités Java les plus utilisées, mais souvent que trop peu maîtrisée : Les collections.
<!--more-->
Cet article s'inscrit dans une série d'articles concernant les collections, leurs utilisations et fonctionnalités dont voici le sommaire :

1. Introduction aux collections Java (vous êtes ici)
2. [Les listes en Java]({% post_url 2025-09-19-Framework-collections-java-list %})
3. [Les ensembles (Set) en Java]({% post_url 2025-09-25-Framework-collections-java-set %})
4. Collections Queue (article à venir)
5. Collections Map (article à venir)
6. Utilisations avancées des collections (article à venir)

## Introduction

Apparue en Java 1.2, l'API Collection propose aux développeurs une manière de stocker et structurer des données quel que soit leur type.

Une collection est simplement un objet qui regroupe plusieurs éléments en une seule unité. Elles sont utilisées pour stocker, structurer, récupérer et manipuler des données. En règle générale, elles représentent des éléments de données qui forment un groupe naturel, par exemple un jeu de cartes, un annuaire téléphonique (une correspondance des noms avec les numéros de téléphone)...

Avec cette définition, il est judicieux de se demander à quoi bon utiliser une collection à la place d'un tableau. La réponse est simple : Les collections sont capables de manipuler un ensemble d'objet dont le nombre n'est pas connu au préalable à la différence d'un tableau qui doit être instancié en connaissant sa taille. Les collections sont ainsi capables d'augmenter dynamiquement leurs tailles au fur et à mesure que des objets y sont insérés. Nous pouvons aussi ajouter qu'un tableau n'est pas *Thread Safe* et ne propose donc pas de protection si deux *Threads* tentent d'accéder en même temps au même tableau.

Retenons alors le principal : Si vous avez besoin de stocker et de manipuler une liste d'objets sans connaître au préalable le nombre de ces objets, vous devrez donc utiliser des collections !

## Présentation du Framework Collections

### Architecture

Le Framework Collections est une architecture présente dans la bibliothèque standard de Java représentant les différentes collections. Ce Framework se constitue de trois éléments principaux :

1. **Les interfaces** : Ensemble de types abstraits représentant et définissant les comportements des différentes familles de collections.
2. **Les implémentations** : Héritant des interfaces, elles correspondent aux implémentations concrètes des collections que vous utiliserez en tant que développeurs.
3. **Les algorithmes** : Ensemble de méthodes permettant de manipuler les collections comme des algorithmes de tri. Ces algorithmes sont dits polymorphiques, car peuvent être utilisés de la même manière quelle que soit la collection utilisée.

Le Framework Collections forme alors une hiérarchie de classes qui est la suivante :

![](/assets/images/2020-11-12-Framework-collections-java-intro/collectionsHierarchy.png)

Dans cette hiérarchie, nous retrouvons deux interfaces principales représentant les grandes familles de collections :

1. **Collection** : Permet de gérer des groupes d'objets.
2. **Map** : Permet de gérer des éléments en paires de type clés/valeurs.

L'interface Collection se sépare ensuite en trois familles distinctes :

1. **List** : Collection d'éléments ordonnés acceptant les doublons.
2. **Set** : Collection d'éléments non ordonnés n'acceptant pas les doublons.
3. **Queue** : Collection qui stocke des éléments dans un certain ordre avant qu'ils ne soient extraits pour traitement.

On remarque ensuite que chaque famille de collections comportent de nombreuses implémentations que ce soit l'**ArrayList** pour l'interface List, le **HashSet** pour l'interface Set ou la **HashMap** pour l'interface Map. Chaque famille de collection possède ses propres caractéristiques, méthodes et cas d'utilisation à respecter que nous détaillerons dans des articles spécifiques à chaque famille.

### Interface Collection

L'interface **Collection** (à ne pas confondre avec la classe **Collections** qui propose des méthodes et algorithme pour les collections) est l'interface définissant le comportement minimal de toutes les collections (hormis les Maps).

Cette interface définie alors les méthodes les plus générales permettant la manipulation des collections : ajout et suppression d'éléments, vérification de la présence d'un élément dans la collection, parcours de la collection... mais également deux constructeurs :

- Un constructeur par défaut qui initialise une collection vide.
- Un constructeur prenant en paramètre une autre collection et qui copie tous les éléments pour créer une nouvelle collection.

Cette interface propose différentes méthodes à ses implémentations telles que les suivantes :

| Méthode                                   | Description                                                                                        |
|-------------------------------------------|----------------------------------------------------------------------------------------------------|
| void add(E e)                             | Ajout de élément de type E                                                                         |
| boolean addAll(Collection<? extends E> c) | Ajoute tous les éléments d'une autre collection du même type d'objets                              |
| void clear()                              | Suppression de tous les éléments                                                                   |
| boolean contains(Object o)                | Vérifie si un élément est présent                                                                  |
| boolean containsAll(Collection<?> c)      | Vérifie si tous les élément d'une autre collection sont présents.                                  |
| boolean isEmpty()                         | Vérifie si la collection en comporte aucun élément                                                 |
| Iterator<E> iterator()                    | Retourne un itérateur pour parcourir les éléments de la collection                                 |
| boolean remove(Object o)                  | Supprime un élément de la collection s'il est présent                                              |
| boolean removeAll(Collection<?> c)        | Supprime tous les élément d'une autre collection s'ils sont présents.                              |
| boolean retainAll(Collection<?> c)        | Filtre la collection et ne laisse que ceux également présents dans l'autre collection en paramètre |
| int size()                                | Retourne le nombre d'éléments dans la collection                                                   |
| Object[] toArray()                        | Retourne un tableau contenant tous les éléments de la collection                                   |
| <E> E[] toArray(E[] a)                    | Retourne un tableau de type E contenant tous les éléments de la collection                         |


Il faut savoir pour ces deux dernières méthodes que la collection et son tableau généré sont indépendants. Toute modification dans la collection n'impactera pas le tableau généré et inversement.

Toutes les familles de collections fournissent alors au minimum ces méthodes et chaque famille de collection vient ensuite ajouter un nouvel ensemble de méthodes qui vont venir spécialiser les futures implémentations.

### Interface Iterator

Parcourir un tableau en java est relativement simple, il suffit d'itérer les éléments de la case 0 à la dernière case du tableau. Pour les collections, cela est un peu plus complexe, car on ne connait pas sa taille et son organisation. Ainsi, l'interface **Iterator** offre une solution pour parcourir facilement les éléments d'une collection qu'importe son implémentation.

Cette interface définie alors les méthodes suivantes :

| Méthode           | Description                                       |
|-------------------|---------------------------------------------------|
| boolean hasNext() | Retourne true s'il reste des éléments à parcourir |
| Object next()     | Retourne le prochain élément à parcourir          |
| void remove()     | Supprime l'élément actuel de l'itérateur          |

Grâce à ces méthodes, il est possible de parcourir tous les éléments d'une collection et de les manipuler. Pour ce faire, il faut utiliser la méthode *iterator()* définie dans l'interface Collection pour obtenir l'itérateur propre à la collection souhaitée et utiliser les méthodes de l'itérateur de la sorte :

```java
void display(Collection maCollection) {
  Iterator<String> iterator = maCollection.iterator();
  while (iterator.hasNext()) {
      String str = iterator.next();
      System.out.println(str);
  }
}
```

Dans cet exemple, nous parcourons une collection *maCollection* contenant des éléments de type String et affichons chaque élément récupéré avec la méthode *next* tant que la méthode *hasNext* retourne true.

## Bonnes pratiques avec les collections

Maintenant que nous avons vu les fondements de l'API Collection et ses interfaces, il est temps de décrire les différentes implémentations existantes. Toutefois, avant cela, il convient poser les bases des bonnes pratiques d'utilisation des collections :

- **Déclaration d'une collection par son interface**

Une des règles primordiale des collections, qui est également une des bases du développement orienté objet en Java est la déclaration d'une variable par son interface. Je m'explique :

Si vous souhaitez déclarer une **ArrayList**, il faut la déclarer par son interface à savoir **List**. Une déclaration correcte est donc la suivante :

```java
List<String> maListe = new ArrayList<String>();
```

Ce principe permet de respecter une notion clé qui est le polymorphisme.

> Le **polymorphisme** est le concept consistant à fournir une interface unique à des entités pouvant avoir différents types.

Ainsi, il sera possible d'utiliser cette *ArrayList* comme étant une *List* sans se poser la question dans le code de son implémentation réelle.

- **Parcours des collections avec le foreach**

Si les itérateurs peuvent vous sembler quelque peu lourds à utiliser, il est également possible de parcourir une collection avec une boucle de type *foreach* de la même manière que les tableaux.

```java
for (String str : maListe) {
	System.out.println(str);
}
```
La boucle *foreach* utilise de manière implicite un itérateur et les méthodes *hasNext()* et *next()* pour parcourir tous les éléments de la collection.

- **Utilisation de la diversité des collections**

Il est souvent assez tentant de créer une *ArrayList* par défaut quand vous avez besoin d'une collection, car c'est celle la plus courante et la plus citée en exemple. Toutefois, il est vivement conseillé d'analyser le jeu de données que vous souhaitez stocker dans une collection, se demander l'utilisation que vous aurez de cette collection et se poser les bonnes questions :

- Ma collection pourra-t-elle contenir des doublons ?
- L'ordre des éléments sera t'il important ?
- La collection doit-elle être *Thread Safe* ?
- Avez-vous besoin d'accéder à des éléments dans la collection et comment (position, identifiant...) ?
- ...

En se posant les bonnes questions, il est possible de savoir exactement quelle collection utiliser pour répondre pleinement à vos besoins.

À l'inverse, si vous êtes certain de connaitre le nombre d'éléments de votre jeu de données, que vous n'aurez pas de problèmes d'accès concurrentiel et que vous souhaitez juste stocker des éléments à des positions fixes et/ou parcourir de manière simple vos éléments, il est alors conseillé de se demander si l'utilisation d'une collection est vraiment nécessaire et utiliser un tableau traditionnel à la place.



## Conclusion

Pouvant paraitre aux premiers abords assez complexe et confus, l'API Collection se révèle finalement très structurée et comprendre son architecture et son implémentation permet alors de maîtriser pleinement les collections.

Le plus important est alors de retenir que cette API est construite sous une forme hiérarchique, grâce à des interfaces au sommet qui définissent des comportements et une structure pour chaque famille de collection, et que chaque implémentation est ainsi modelée par cette hiérarchie d'interfaces.

Il est maintenant temps de s'intéresser aux collections concrètes que représentent les implémentations de cette API grâce aux articles suivants :

- [Les listes en Java]({% post_url 2025-09-19-Framework-collections-java-list %})
- [Les ensembles (Set) en Java]({% post_url 2025-09-25-Framework-collections-java-set %})
- Collections Queue (article à venir)
- Collections Map (article à venir)
- Utilisations avancées des collections (article à venir)

### Pour aller plus loin

 - [Collection - Javadoc Java 17](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Collection.html)
 - [Iterator - Javadoc Java 17](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Iterator.html)
