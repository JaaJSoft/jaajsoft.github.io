---
layout: article
title: "Spring : Comment utiliser les application properties"
tags:
    - java
    - spring
    - tutoriel

author: Rémi Lecouillard
---

Dans ce tutoriel, vous allez apprendre à définir des properties spring et à les utiliser dans votre code Java.
Ce tutoriel suppose que vous savez déjà *set-up* un projet avec Spring boot et que vous avez quelques bases de programmation Java.

## C'est quoi les _application properties_ Spring ?

Plus communément appelé _properties_, elles sont des valeurs accessibles dans toute
votre application.

Spring les utilise pour de nombreux paramètres, la plupart possèdent des valeurs par défaut mais que vous pouvez aussi redéfinir par vous-même. Vous pouvez retrouver la liste complète de ces paramètres [ici](https://docs.spring.io/spring-boot/docs/current/reference/html/appendix-application-properties.html).

Vous pouvez également créer vos propres _properties_ pour vos besoins spécifiques.

## Définir une _property_ Spring

### Fichier par défaut

Que ce soit pour définir les différents paramètres de spring ou vos propres
properties, Spring recherche par défaut les properties dans le fichier
`application.properties` ou `application.yaml`

Ces fichiers sont recherchés dans les dossiers suivants :

* La racine du classpath
* Le package /config du classpath
* Le repertoire courant
* Le sous repertoire /config du repertoire courant
* Les sous repertoires directes du sous repertoire /config

Si vous utilisez le _Standard Directory Layout_, que se soit avec Maven ou Gradle,
les fichiers sont généralement mis dans `src/main/resources`. Puisqu'on peut y accéder depuis le _classpath_.

### Définir ses propres fichiers

Si vous voulez accéder à des properties définies dans un fichier comportant un
autre nom, c'est très simple. Il suffit d'utiliser l'annotation `@PropertySource`
comme ci dessous :

```java
@Configuration
@PropertySource("classpath:foo.properties")
@PropertySource("classpath:toto.properties")
public class PropertiesWithJavaConfig {
    //...
}
```

Il est impératif de l'utiliser avec l'annotation `@Configuration`.

Comme vous avez pu le remarquer la même annotation est défini deux fois. On peut
définir l'annotation autant de fois qu'on le souhaite pour définir autant de fichiers.
Une autre façon de définir plusieurs fichiers est la suivante :

```java
@PropertySources({
    @PropertySource("classpath:foo.properties"),
    @PropertySource("classpath:toto.properties")
})
public class PropertiesWithJavaConfig {
    //...
}
```

Il est intéressant de noter qu'en cas de conflit de nom, c'est la dernière _property_
lue qui est utilisée.

Si vous voulez configurer les noms de vos fichiers, c'est possible. Pour cela
il faut utiliser les placeholders.

```java
@PropertySource({
  "classpath:persistence-${db.provider:mysql}.properties"
})
public class PropertiesWithJavaConfig {
    //...
}
```

Dans ce cas, si la property `db.provider` a été préalablement défini par mongodb par
exemple, le fichier `persistence-mongodb.properties` sera chargé. Si elle n'est pas
définie ce sera la valeur après le ':' qui sera utilisée. À savoir qu'utiliser le
':' est optionnel, mais si la _property_ n'est jamais déclarée une exception sera levée.

### Les différents formats de fichier

Il existe deux formats de fichiers possibles pour définir les _properties_. Soit le format Properties comme ci dessous :

```properties
app.name=MyApp
app.description=${app.name} is a Spring Boot application
```

Soit le format Yaml, comme ci-dessous :

```yaml
app:
  name: "MyApp"
  description: "${app.name} is a Spring Boot application"
```

Attention, les fichiers Yaml ne sont pas disponibles avec l'annotation `@PropertySource`.

#### Property Placeholders

Comme vous l'avez peut-être remarqué au-dessus, nous avons utilisé `${app.name}`.
Cette syntaxe permet de référé à une autre _property_.

## Utiliser les _properties_ en Java

Il existe essentiellement trois manières différentes d'accéder aux properties depuis le code Java.

### L'annotation @Value

On peut accéder à une property très facilement en l'injectant via l'annotation @Value.
Ici par exemple si on veut accéder à la property keycloak.url il faudra marquer :

```java
@Value( "${keycloak.url}" )
private String keycloakUrl;
```

On peut également définir une valeur par défaut de cette façon :

```java
@Value( "${keycloak.url:UnUrlParDéfaut}" )
private String keycloakUrl;
```

### L'objet Environment

Il est aussi possible d'injecter un objet environnement qui vous permet ensuite
d'accéder à n'importe quelle _property_ via une méthode comme suit :

```java
@Autowired
private Environment env;
...
keycloakUrl = env.getProperty("keycloak.url");
```

### Mapping d'objet java

Dans le cas de _properties_ groupées ensemble, on peut utiliser l'annotation @ConfigurationProperties pour les mapper avec un objet Java.

Prenons l'exemple de _properties_ pour configurer la connection à une base de données :

```properties
database.url=jdbc:postgresql:/localhost:5432/instance
database.username=foo
database.password=bar
```

Il suffit ensuite d'utiliser l'annotation sur une classe pour les *mapper*.

```java
@ConfigurationProperties(prefix = "database")
public class Database {
    String url;
    String username;
    String password;

    // standard getters and setters
}
```

## Conclusion

Comme nous l'avons vu Spring offre un panel de possibilités assez large pour
déclarer et utiliser très facilement les _properties_ à votre convenance selon
les besoins de votre application.

## Voir aussi

- [La doc de spring sur la configuration externe](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-external-config-files)
