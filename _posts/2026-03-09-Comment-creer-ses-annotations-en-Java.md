---
layout: article
title: "Comment créer ses annotations en Java"
tags:
  - java
  - annotations
  - reflection
author: Pierre Chopinet
---

On utilise des annotations tous les jours en Java (`@Override`, `@Autowired`, `@GetMapping`...), mais on en écrit rarement. Déclarer la sienne ne prend que quelques lignes : le vrai travail se trouve dans le code qui la lit, soit à l'exécution par réflexion, soit pendant la compilation avec un processeur d'annotations. Nous allons voir les deux.
<!--more-->

Dans cet article :
- Déclarer une annotation
- Les méta-annotations
- Les paramètres d'une annotation
- Lire les annotations par réflexion
- Un mini-framework de validation
- Tracer les appels avec un proxy
- Vérifier le code à la compilation

Pré-requis : Java 16 ou plus récent pour certains exemples (`instanceof` avec pattern matching, `Stream.toList()`). Les exemples ont été testés avec Java 21, et le processeur d'annotations aussi avec Java 25.

## Déclarer une annotation

Une annotation est une métadonnée attachée au code : elle ne change rien au comportement du programme tant que personne ne la lit. Celles que l'on croise le plus souvent sont lues par le compilateur :

```java
@Override  // Vérifie que la méthode redéfinit bien une méthode parente
public String toString() {
    return "exemple";
}

@Deprecated(since = "17")  // Marque un élément comme obsolète
public void ancienneMethode() {}

@SuppressWarnings("unchecked")  // Supprime un avertissement du compilateur
public void methodeAvecCast() {}
```

Les nôtres seront lues par notre propre code à l'exécution, ou par un outil branché sur le compilateur. Une annotation se déclare avec le mot-clé `@interface` :

```java
public @interface MonAnnotation {
}
```

Et voilà, `@MonAnnotation` est utilisable. Derrière ce mot-clé, le compilateur génère une interface qui étend `java.lang.annotation.Annotation`. On ne l'instancie jamais avec `new` : c'est l'API de réflexion qui fournit les instances quand on lit l'annotation. Pour qu'elle serve à quelque chose, il reste à préciser où on peut la placer et jusqu'à quand elle est conservée, avec des méta-annotations.

## Les méta-annotations

Les méta-annotations sont des annotations qui s'appliquent à la déclaration d'une autre annotation.

### @Retention : jusqu'à quand l'annotation est conservée

```java
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;

// Disponible uniquement dans le code source (supprimée à la compilation)
@Retention(RetentionPolicy.SOURCE)
public @interface Todo {}

// Conservée dans le bytecode, mais pas accessible à l'exécution
@Retention(RetentionPolicy.CLASS)
public @interface GeneratedCode {}

// Conservée dans le bytecode ET accessible à l'exécution via la réflexion
@Retention(RetentionPolicy.RUNTIME)
public @interface Auditable {}
```

| Politique | Dans le code source | Dans le fichier .class | Lisible par réflexion |
|-----------|:-------------------:|:----------------------:|:---------------------:|
| `SOURCE`  | Oui                 | Non                    | Non                   |
| `CLASS`   | Oui                 | Oui                    | Non                   |
| `RUNTIME` | Oui                 | Oui                    | Oui                   |

Sans `@Retention`, la politique par défaut est `CLASS` : l'annotation est bien écrite dans le `.class`, mais `getAnnotation()` renvoie `null` à l'exécution. Une annotation destinée à être lue par réflexion doit donc être en `RUNTIME`, c'est la première chose à vérifier quand une annotation maison semble ignorée.

### @Target : où l'annotation peut être placée

```java
import java.lang.annotation.Target;
import java.lang.annotation.ElementType;

@Target(ElementType.METHOD)  // Uniquement sur les méthodes
public @interface LogExecution {}

@Target({ElementType.FIELD, ElementType.PARAMETER})  // Sur les champs et paramètres
public @interface NotEmpty {}
```

Les valeurs possibles de `ElementType` :

| Valeur             | Cible                                               |
|--------------------|-----------------------------------------------------|
| `TYPE`             | Classe, interface (annotations comprises), enum, record |
| `FIELD`            | Champ, y compris les constantes d'un enum           |
| `METHOD`           | Méthode                                             |
| `PARAMETER`        | Paramètre de méthode                                |
| `CONSTRUCTOR`      | Constructeur                                        |
| `LOCAL_VARIABLE`   | Variable locale                                     |
| `ANNOTATION_TYPE`  | Déclaration d'une autre annotation                  |
| `PACKAGE`          | Déclaration de package                              |
| `TYPE_PARAMETER`   | Paramètre de type générique (Java 8+)               |
| `TYPE_USE`         | Utilisation d'un type (Java 8+)                     |
| `MODULE`           | Déclaration de module (Java 9+)                     |
| `RECORD_COMPONENT` | Composant de record (Java 16+)                      |

Sans `@Target`, l'annotation peut être placée sur n'importe quelle déclaration. Avec, le compilateur refuse tout autre emplacement avec l'erreur `annotation interface not applicable to this kind of declaration`. `ANNOTATION_TYPE` sert à écrire des méta-annotations : c'est avec `@Target(ElementType.ANNOTATION_TYPE)` que sont déclarées `@Retention` et `@Target` elles-mêmes dans le JDK.

### @Documented

`@Documented` indique que l'annotation fait partie du contrat public des éléments annotés : la Javadoc d'une méthode annotée `@ApiEndpoint` affichera l'annotation, ce qui n'est pas le cas pour une annotation sans `@Documented`.

```java
import java.lang.annotation.Documented;

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ApiEndpoint {
    String value();
}
```

### @Inherited

`@Inherited` permet à une sous-classe d'hériter de l'annotation posée sur sa classe parente :

```java
import java.lang.annotation.Inherited;

@Inherited
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Cacheable {}

@Cacheable
public class BaseService {}

// ChildService hérite de @Cacheable
public class ChildService extends BaseService {}
```

`ChildService` n'est pas annotée, mais `ChildService.class.isAnnotationPresent(Cacheable.class)` renvoie `true`, car la recherche remonte vers la classe parente. Par contre, `getDeclaredAnnotation(Cacheable.class)` renvoie `null`, cette méthode ne regardant que la classe elle-même. `@Inherited` ne fonctionne que pour les annotations de classes : rien n'est hérité pour les méthodes, ni depuis les interfaces implémentées.

Attention à ne pas confondre avec un héritage entre annotations, qui n'existe pas : `@interface Fille extends Base` est refusé par le compilateur (`'extends' not allowed for @interfaces`).

### @Repeatable

Par défaut, une annotation ne peut apparaître qu'une fois sur un même élément. `@Repeatable` (Java 8+) lève cette limite, à condition de déclarer une annotation conteneur :

```java
import java.lang.annotation.Repeatable;

@Repeatable(Roles.class)
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Role {
    String value();
}

// Annotation conteneur (obligatoire)
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Roles {
    Role[] value();
}

// Utilisation : plusieurs @Role sur la même classe
@Role("ADMIN")
@Role("USER")
public class AdminController {}
```

À la compilation, les deux `@Role` sont rangées dans un `@Roles`. Du coup, `AdminController.class.getAnnotation(Role.class)` renvoie `null`. Pour les lire, on utilise `getAnnotationsByType`, qui fonctionne qu'il y ait une ou plusieurs annotations :

```java
for (Role role : AdminController.class.getAnnotationsByType(Role.class)) {
    System.out.println(role.value());
}
```

```
ADMIN
USER
```

## Les paramètres d'une annotation

Une annotation peut déclarer des éléments, qui s'écrivent comme des méthodes sans paramètre, avec une valeur par défaut optionnelle :

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface RateLimit {
    int maxRequests();               // Obligatoire (pas de valeur par défaut)
    int windowSeconds() default 60;  // Optionnel (60 secondes par défaut)
    String message() default "Trop de requêtes";
}

// Utilisation
@RateLimit(maxRequests = 100)
public void getUsers() {}

@RateLimit(maxRequests = 10, windowSeconds = 30, message = "Limite atteinte")
public void createUser() {}
```

Un élément sans valeur par défaut est obligatoire : `@RateLimit` tout seul ne compile pas (`annotation @RateLimit is missing a default value for the element 'maxRequests'`).

Le type d'un élément est limité à :
- un type primitif (`int`, `long`, `double`, `boolean`...)
- `String`
- `Class<?>` ou `Class<? extends T>`
- un enum
- une autre annotation
- un tableau d'un des types précédents

Un `Integer` ou une `List<String>` donnent l'erreur `invalid type for annotation interface element`. Les valeurs doivent en plus être des constantes connues à la compilation, ce qui exclut `null` : `String value() default null;` ne compile pas (`element value must be a constant expression`). On utilise à la place une valeur conventionnelle, comme la chaîne vide dans l'exemple `@JsonField` plus bas.

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Entity {
    String table();
    String schema() default "public";
    Class<?>[] listeners() default {};
    // CascadeType est ici un enum maison (à ne pas confondre avec celui de JPA)
    CascadeType cascade() default CascadeType.NONE;
}
```

Quand l'annotation n'a qu'un élément nommé `value`, son nom peut être omis à l'utilisation : `@Column("user_name")` revient à écrire `@Column(value = "user_name")`. Ça fonctionne aussi si les autres éléments ont une valeur par défaut :

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface Column {
    String value();
    boolean nullable() default true;
}

@Column("email")  // OK : nullable prend la valeur par défaut
private String email;
```

## Lire les annotations par réflexion

Les annotations en `RetentionPolicy.RUNTIME` se lisent avec l'API de réflexion. `isAnnotationPresent()` indique si un élément porte une annotation :

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Transactional {}

public class UserService {
    @Transactional
    public void save(String user) {}

    public void find(String user) {}
}

// Lecture via réflexion
Method saveMethod = UserService.class.getMethod("save", String.class);
Method findMethod = UserService.class.getMethod("find", String.class);

System.out.println(saveMethod.isAnnotationPresent(Transactional.class));  // true
System.out.println(findMethod.isAnnotationPresent(Transactional.class));  // false
```

`getAnnotation()` renvoie l'annotation elle-même, ce qui donne accès à ses paramètres :

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Retry {
    int maxAttempts() default 3;
    long delayMs() default 1000;
}

public class RemoteService {
    @Retry(maxAttempts = 5, delayMs = 2000)
    public String callApi() { return ""; }
}

// Lecture des valeurs
Method method = RemoteService.class.getMethod("callApi");
Retry retry = method.getAnnotation(Retry.class);

System.out.println(retry.maxAttempts());  // 5
System.out.println(retry.delayMs());      // 2000
```

Pour trouver toutes les méthodes annotées d'une classe, on filtre `getDeclaredMethods()` :

```java
public static List<Method> findAnnotatedMethods(Class<?> clazz,
                                                 Class<? extends Annotation> annotation) {
    return Arrays.stream(clazz.getDeclaredMethods())
        .filter(m -> m.isAnnotationPresent(annotation))
        .toList();
}

// Utilisation
List<Method> transactionalMethods = findAnnotatedMethods(UserService.class, Transactional.class);
transactionalMethods.forEach(m -> System.out.println(m.getName()));  // save
```

Attention, `getDeclaredMethods()` ne renvoie pas les méthodes dans un ordre garanti. Avec deux méthodes annotées, `save` puis `delete`, Java 21 renvoie ici `delete` en premier et Java 25 `save`. Si l'ordre compte, il faut trier le résultat.

Les champs se lisent de la même façon. Voici une sérialisation très simplifiée, qui ne garde que les champs annotés `@JsonField` et permet de renommer la clé :

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface JsonField {
    String value() default "";
}

public class User {
    @JsonField("user_name")
    private String name;

    @JsonField
    private String email;

    private int age;  // Pas annoté

    public User(String name, String email, int age) {
        this.name = name;
        this.email = email;
        this.age = age;
    }
}

// Sérialisation simple basée sur les annotations
public static Map<String, Object> serialize(Object obj) throws IllegalAccessException {
    Map<String, Object> result = new LinkedHashMap<>();

    for (Field field : obj.getClass().getDeclaredFields()) {
        JsonField annotation = field.getAnnotation(JsonField.class);
        if (annotation != null) {
            field.setAccessible(true);
            String key = annotation.value().isEmpty() ? field.getName() : annotation.value();
            result.put(key, field.get(obj));
        }
    }

    return result;
}
```

```java
System.out.println(serialize(new User("Alice", "alice@mail.com", 30)));
```

```
{user_name=Alice, email=alice@mail.com}
```

`setAccessible(true)` permet de lire les champs `private`. Les champs sortent ici dans l'ordre de leur déclaration, mais comme pour les méthodes, la Javadoc de `getDeclaredFields()` ne le garantit pas.

Ces appels de réflexion ont un coût. Ce n'est pas gênant pour du code exécuté une fois au démarrage, mais si la lecture des annotations se fait à chaque appel, mieux vaut garder le résultat de côté, par exemple dans une `Map` par classe.

## Un mini-framework de validation

Assemblons tout ça : trois annotations de validation posées sur les champs d'un formulaire, et un validateur qui les lit. C'est le principe de Bean Validation (`@NotNull`, `@Size`, `@Min` de Jakarta Validation), en beaucoup plus simple.

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface NotNull {
    String message() default "Le champ ne doit pas être null";
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface MinLength {
    int value();
    String message() default "Longueur minimale non respectée";
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface Range {
    int min();
    int max();
    String message() default "Valeur hors limites";
}
```

Le formulaire annoté :

```java
public class UserForm {
    @NotNull
    @MinLength(3)
    private String name;

    @NotNull
    @MinLength(5)
    private String email;

    @Range(min = 18, max = 120)
    private int age;

    public UserForm(String name, String email, int age) {
        this.name = name;
        this.email = email;
        this.age = age;
    }
}
```

Le validateur parcourt les champs et, pour chaque annotation présente, vérifie la valeur et ajoute un message en cas d'erreur :

```java
public class Validator {

    public static List<String> validate(Object obj) {
        List<String> errors = new ArrayList<>();

        for (Field field : obj.getClass().getDeclaredFields()) {
            field.setAccessible(true);
            Object value;
            try {
                value = field.get(obj);
            } catch (IllegalAccessException e) {
                continue;
            }

            // Vérification @NotNull
            if (field.isAnnotationPresent(NotNull.class) && value == null) {
                NotNull ann = field.getAnnotation(NotNull.class);
                errors.add(field.getName() + " : " + ann.message());
                continue;
            }

            // Vérification @MinLength
            if (field.isAnnotationPresent(MinLength.class) && value instanceof String s) {
                MinLength ann = field.getAnnotation(MinLength.class);
                if (s.length() < ann.value()) {
                    errors.add(field.getName() + " : " + ann.message()
                        + " (minimum " + ann.value() + ")");
                }
            }

            // Vérification @Range
            if (field.isAnnotationPresent(Range.class) && value instanceof Number n) {
                Range ann = field.getAnnotation(Range.class);
                int intValue = n.intValue();
                if (intValue < ann.min() || intValue > ann.max()) {
                    errors.add(field.getName() + " : " + ann.message()
                        + " [" + ann.min() + "-" + ann.max() + "]");
                }
            }
        }

        return errors;
    }
}
```

On le teste avec un formulaire valide, puis avec un formulaire qui enfreint les trois règles :

```java
UserForm valid = new UserForm("Alice", "alice@mail.com", 30);
System.out.println(Validator.validate(valid));

UserForm invalid = new UserForm("Al", null, 15);
Validator.validate(invalid).forEach(System.out::println);
```

Ce qui donne :

```
[]
name : Longueur minimale non respectée (minimum 3)
email : Le champ ne doit pas être null
age : Valeur hors limites [18-120]
```

## Tracer les appels avec un proxy

Un proxy dynamique (`java.lang.reflect.Proxy`) intercepte tous les appels faits à travers une interface. Combiné à une annotation, il permet d'ajouter un comportement aux seules méthodes annotées. Ici, `@Audited` écrit une ligne de log à chaque appel :

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Audited {
    String action() default "";
}
```

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.time.LocalDateTime;
import java.util.Arrays;

public class AuditProxy implements InvocationHandler {
    private final Object target;

    private AuditProxy(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // Chercher l'annotation sur la méthode de la classe cible
        Method targetMethod = target.getClass().getMethod(method.getName(), method.getParameterTypes());

        if (targetMethod.isAnnotationPresent(Audited.class)) {
            Audited audited = targetMethod.getAnnotation(Audited.class);
            String action = audited.action().isEmpty() ? method.getName() : audited.action();
            System.out.printf("[AUDIT] %s | action=%s | args=%s%n",
                LocalDateTime.now(), action, Arrays.toString(args));
        }

        try {
            return method.invoke(target, args);
        } catch (InvocationTargetException e) {
            throw e.getCause();  // l'exception levée par la méthode cible
        }
    }

    @SuppressWarnings("unchecked")
    public static <T> T create(T target, Class<T> iface) {
        return (T) Proxy.newProxyInstance(
            iface.getClassLoader(),
            new Class<?>[]{iface},
            new AuditProxy(target)
        );
    }
}
```

L'annotation est cherchée sur la méthode de la classe cible, car `method` est celle de l'interface, qui n'est pas annotée. Le `try/catch` autour de `method.invoke()` a aussi son importance : sans lui, une exception levée par le service (une `IllegalArgumentException` par exemple) n'arriverait pas telle quelle chez l'appelant, mais emballée dans une `UndeclaredThrowableException`.

Un proxy ne sait implémenter que des interfaces, il nous faut donc une interface pour le service :

```java
public interface OrderService {
    void placeOrder(String product, int quantity);
    String getOrder(String orderId);
}

public class OrderServiceImpl implements OrderService {
    @Audited(action = "PLACE_ORDER")
    public void placeOrder(String product, int quantity) {
        System.out.println("Commande passée : " + product + " x" + quantity);
    }

    @Audited
    public String getOrder(String orderId) {
        return "Order-" + orderId;
    }
}

// Création du proxy audité
OrderService service = AuditProxy.create(new OrderServiceImpl(), OrderService.class);

service.placeOrder("Laptop", 2);
service.getOrder("ABC");
```

On obtient quelque chose comme :

```
[AUDIT] 2026-10-07T20:37:33.565321910 | action=PLACE_ORDER | args=[Laptop, 2]
Commande passée : Laptop x2
[AUDIT] 2026-10-07T20:37:33.584695265 | action=getOrder | args=[ABC]
```

Ce mécanisme a une limite : le proxy ne voit que les appels qui passent par lui. Si `placeOrder` appelait `getOrder` en interne, cet appel ne serait pas audité. Spring s'appuie sur des proxys du même genre (ou sur des sous-classes générées quand la classe n'implémente pas d'interface) pour traiter `@Transactional` ou `@Cacheable`, avec la même limite dans sa configuration par défaut : un appel interne à une méthode `@Cacheable` ne passe pas par le cache.

## Vérifier le code à la compilation

Les exemples précédents lisent les annotations à l'exécution. Un processeur d'annotations, lui, est appelé par `javac` pendant la compilation, à travers l'API `javax.annotation.processing`. Il peut émettre des erreurs ou des avertissements, et générer de nouveaux fichiers (sources, ressources), mais il ne peut pas modifier les classes existantes. C'est comme ça que MapStruct ou Dagger génèrent leur code. Lombok, qui ajoute des méthodes aux classes elles-mêmes, démarre aussi comme un processeur d'annotations, mais s'appuie ensuite sur des API internes de `javac`.

Voici un processeur qui vérifie que les classes annotées `@Builder` ont au moins un champ :

```java
package com.example;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.SOURCE)
@Target(ElementType.TYPE)
public @interface Builder {}
```

```java
package com.example;

import javax.annotation.processing.*;
import javax.lang.model.SourceVersion;
import javax.lang.model.element.*;
import javax.tools.Diagnostic;
import java.util.Set;

@SupportedAnnotationTypes("com.example.Builder")
public class BuilderProcessor extends AbstractProcessor {

    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations,
                           RoundEnvironment roundEnv) {

        for (Element element : roundEnv.getElementsAnnotatedWith(Builder.class)) {
            if (element.getKind() != ElementKind.CLASS) {
                processingEnv.getMessager().printMessage(
                    Diagnostic.Kind.ERROR,
                    "@Builder ne peut être utilisé que sur des classes",
                    element
                );
                continue;
            }

            TypeElement typeElement = (TypeElement) element;
            long fieldCount = typeElement.getEnclosedElements().stream()
                .filter(e -> e.getKind() == ElementKind.FIELD)
                .count();

            if (fieldCount == 0) {
                processingEnv.getMessager().printMessage(
                    Diagnostic.Kind.ERROR,
                    "@Builder requiert au moins un champ",
                    element
                );
            }
        }

        return true;
    }
}
```

La rétention `SOURCE` suffit pour `@Builder` : le processeur travaille sur le code source, l'annotation n'a pas besoin d'aller plus loin. `@SupportedAnnotationTypes` indique l'annotation traitée, et `process()` renvoie `true` pour la réclamer : les autres processeurs ne la recevront pas.

Pour la version de Java supportée, on trouve souvent `@SupportedSourceVersion(SourceVersion.RELEASE_17)`. Le problème, c'est que `javac` affiche alors un avertissement dès qu'on compile pour une version plus récente : `Supported source version 'RELEASE_17' from annotation processor 'com.example.BuilderProcessor' less than -source '21'`. Redéfinir `getSupportedSourceVersion()` pour renvoyer `SourceVersion.latestSupported()` évite ce message.

### Enregistrer et lancer le processeur

Le processeur est découvert par `javac` grâce au fichier `META-INF/services/javax.annotation.processing.Processor`, qui contient son nom qualifié :

```
com.example.BuilderProcessor
```

Avec les modules Java, on utilise à la place la directive `provides` dans `module-info.java` :

```java
provides javax.annotation.processing.Processor
    with com.example.BuilderProcessor;
```

Le processeur doit être compilé avant le code qui l'utilise, en général dans son propre jar. Ensuite, on le passe à `javac` avec `-processorpath` :

```bash
# Le processeur (src/ contient com/example/ et META-INF/services/)
javac -d out src/com/example/Builder.java src/com/example/BuilderProcessor.java
cp -r src/META-INF out/
jar cf builder-processor.jar -C out .

# Le code qui utilise @Builder
javac -processorpath builder-processor.jar -cp builder-processor.jar -d classes app/com/example/*.java
```

Avec une classe `Vide` annotée `@Builder` mais sans champ, la compilation échoue :

```
app/com/example/Vide.java:4: error: @Builder requiert au moins un champ
public class Vide {
       ^
1 error
```

Attention à l'option `-processorpath`. Jusqu'à Java 22, `javac` exécutait aussi les processeurs trouvés sur le simple classpath (Java 21 affiche une note qui annonce que ça va changer). Depuis Java 23, ce n'est plus le cas : sans `-processorpath` (ou `--processor-path`), `-processor` ou `-proc:full`, le processeur est ignoré, sans aucun message.

Avec Maven, le processeur se déclare dans `annotationProcessorPaths` (testé avec le `maven-compiler-plugin` 3.13.0). Il doit aussi être déclaré comme dépendance, en scope `provided`, pour que `@Builder` soit visible dans le code :

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-compiler-plugin</artifactId>
  <version>3.13.0</version>
  <configuration>
    <annotationProcessorPaths>
      <path>
        <groupId>com.example</groupId>
        <artifactId>builder-processor</artifactId>
        <version>1.0</version>
      </path>
    </annotationProcessorPaths>
  </configuration>
</plugin>
```

Voilà, vous savez maintenant déclarer vos annotations et les exploiter, à l'exécution comme à la compilation.

## Voir aussi

- [Comment ajouter du cache à une application Spring Boot]({% post_url 2025-11-08-Comment-ajouter-du-cache-a-une-application-Spring-Boot %})
- [Records en Java : simplifier vos DTOs]({% post_url 2026-01-10-Records-en-Java-simplifier-vos-DTOs %})
- [Pattern matching en Java moderne]({% post_url 2025-10-23-Pattern-matching-en-Java-moderne %})
- [Tutoriel Oracle sur les annotations](https://docs.oracle.com/javase/tutorial/java/annotations/)
- [Java Language Specification : les interfaces d'annotation](https://docs.oracle.com/javase/specs/jls/se21/html/jls-9.html#jls-9.6)
- [Javadoc du package javax.annotation.processing](https://docs.oracle.com/en/java/javase/21/docs/api/java.compiler/javax/annotation/processing/package-summary.html)
