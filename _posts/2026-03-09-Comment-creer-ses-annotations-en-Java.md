---
layout: article
title: "Comment créer ses annotations en Java"
tags:
  - java
  - annotations
  - reflection
author: Pierre Chopinet
---

Les annotations font partie intégrante de Java moderne : `@Override`, `@Autowired`, `@GetMapping`. On les utilise quotidiennement, mais comment créer les nôtres ? Créer ses propres annotations permet d'ajouter des métadonnées à son code et d'automatiser des comportements récurrents.
<!--more-->

Dans cet article, vous découvrirez :
- Ce qu'est une annotation et comment elle fonctionne en interne
- Les méta-annotations qui contrôlent le comportement des annotations (`@Retention`, `@Target`)
- Comment déclarer une annotation avec ou sans paramètres
- Comment lire les annotations à l'exécution via la réflexion
- Comment créer un processeur d'annotations à la compilation
- Des cas d'usage concrets : validation, audit, injection

Pré-requis : Java 8+ pour les bases, Java 17+ recommandé pour les exemples avancés.

---

## Qu'est-ce qu'une annotation ?

Une annotation est une forme de **métadonnée** attachée au code. Elle ne modifie pas directement le comportement du programme, mais elle fournit des informations exploitables par le compilateur, les outils de build ou le runtime.

### Les annotations que vous connaissez déjà

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

Ces annotations sont traitées par le compilateur. Mais rien ne vous empêche de créer les vôtres, traitées soit à la compilation, soit à l'exécution.

### Anatomie d'une déclaration d'annotation

Une annotation se déclare avec `@interface` (et non `class` ou `interface`) :

```java
public @interface MonAnnotation {
}
```

C'est tout. Vous avez créé une annotation utilisable avec `@MonAnnotation`. Mais pour qu'elle soit réellement utile, il faut la configurer avec des **méta-annotations**.

---

## Les méta-annotations

Les méta-annotations sont des annotations qui s'appliquent à d'autres annotations. Elles définissent **où**, **quand** et **comment** votre annotation se comporte.

### @Retention : durée de vie de l'annotation

`@Retention` détermine jusqu'à quand l'annotation est conservée :

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

| Politique | Présente dans le source | Présente dans le bytecode | Accessible via réflexion |
|-----------|:-----------------------:|:-------------------------:|:------------------------:|
| `SOURCE`  | Oui                     | Non                       | Non                      |
| `CLASS`   | Oui                     | Oui                       | Non                      |
| `RUNTIME` | Oui                     | Oui                       | Oui                      |

> La politique par défaut est `CLASS`. Pour la plupart des cas d'usage custom, vous utiliserez `RUNTIME`.

### @Target : où l'annotation peut être placée

`@Target` restreint les emplacements où votre annotation est autorisée :

```java
import java.lang.annotation.Target;
import java.lang.annotation.ElementType;

@Target(ElementType.METHOD)  // Uniquement sur les méthodes
public @interface LogExecution {}

@Target({ElementType.FIELD, ElementType.PARAMETER})  // Sur les champs et paramètres
public @interface NotEmpty {}
```

Les valeurs possibles de `ElementType` :

| Valeur             | Cible                                |
|--------------------|--------------------------------------|
| `TYPE`             | Classe, interface, enum, record      |
| `FIELD`            | Champ (attribut)                     |
| `METHOD`           | Méthode                             |
| `PARAMETER`        | Paramètre de méthode                |
| `CONSTRUCTOR`      | Constructeur                        |
| `LOCAL_VARIABLE`   | Variable locale                     |
| `ANNOTATION_TYPE`  | Autre annotation (méta-annotation)  |
| `PACKAGE`          | Déclaration de package              |
| `TYPE_PARAMETER`   | Paramètre de type générique         |
| `TYPE_USE`         | Utilisation de type (Java 8+)       |
| `RECORD_COMPONENT` | Composant de record (Java 16+)      |

### @Documented

`@Documented` indique que l'annotation doit apparaître dans la Javadoc générée :

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

`@Inherited` permet aux sous-classes d'hériter automatiquement de l'annotation de leur classe parente :

```java
import java.lang.annotation.Inherited;

@Inherited
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Cacheable {}

@Cacheable
public class BaseService {}

// ChildService hérite automatiquement de @Cacheable
public class ChildService extends BaseService {}
```

> `@Inherited` ne fonctionne qu'avec les annotations sur les classes, pas sur les méthodes ni les interfaces.

### @Repeatable (Java 8+)

`@Repeatable` autorise l'utilisation multiple de la même annotation sur un même élément :

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

---

## Annotations avec paramètres

Les annotations peuvent déclarer des **éléments** (paramètres) avec des valeurs par défaut optionnelles.

### Syntaxe des éléments

Les éléments d'une annotation ressemblent à des méthodes abstraites sans paramètres :

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

### Types autorisés

Les éléments d'une annotation sont limités aux types suivants :
- Types primitifs (`int`, `long`, `double`, `boolean`)
- `String`
- `Class<?>` ou `Class<? extends T>`
- Enums
- Autres annotations
- Tableaux des types ci-dessus

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Entity {
    String table();
    String schema() default "public";
    Class<?>[] listeners() default {};
    CascadeType cascade() default CascadeType.NONE;
}
```

### L'élément spécial value()

Si votre annotation ne possède qu'un seul élément nommé `value`, le nom peut être omis à l'utilisation :

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface Column {
    String value();
}

// Les deux écritures sont équivalentes
@Column(value = "user_name")
private String name;

@Column("user_name")
private String name;
```

Cela fonctionne aussi si les autres éléments ont des valeurs par défaut :

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

---

## Lire les annotations à l'exécution (réflexion)

Les annotations avec `RetentionPolicy.RUNTIME` sont accessibles via l'API de réflexion de Java.

### Vérifier la présence d'une annotation

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

### Récupérer les valeurs des éléments

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

### Scanner toutes les méthodes annotées d'une classe

```java
public static List<Method> findAnnotatedMethods(Class<?> clazz,
                                                 Class<? extends Annotation> annotation) {
    return Arrays.stream(clazz.getDeclaredMethods())
        .filter(m -> m.isAnnotationPresent(annotation))
        .toList();
}

// Utilisation
List<Method> transactionalMethods = findAnnotatedMethods(UserService.class, Transactional.class);
transactionalMethods.forEach(m -> System.out.println(m.getName()));
```

### Lire les annotations sur les champs

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
}

// Sérialisation simple basée sur les annotations
public static Map<String, Object> serialize(Object obj) throws Exception {
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

---

## Cas pratique : créer une annotation de validation

Mettons en pratique ce que nous avons vu en créant un mini-framework de validation basé sur les annotations.

### Définir les annotations

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

### Le modèle annoté

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

### Le validateur

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

### Utilisation

```java
UserForm valid = new UserForm("Alice", "alice@mail.com", 30);
List<String> errors1 = Validator.validate(valid);
System.out.println(errors1);  // []

UserForm invalid = new UserForm("Al", null, 15);
List<String> errors2 = Validator.validate(invalid);
// [name : Longueur minimale non respectée (minimum 3),
//  email : Le champ ne doit pas être null,
//  age : Valeur hors limites [18-120]]
```

---

## Cas pratique : annotation d'audit sur les méthodes

Créons une annotation `@Audited` qui loggue automatiquement les appels de méthodes via un proxy dynamique.

### L'annotation

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Audited {
    String action() default "";
}
```

### Le handler de proxy

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.time.LocalDateTime;

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

        return method.invoke(target, args);
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

### Utilisation du proxy

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
// [AUDIT] 2026-03-09T10:30:00 | action=PLACE_ORDER | args=[Laptop, 2]
// Commande passée : Laptop x2

service.getOrder("ABC");
// [AUDIT] 2026-03-09T10:30:01 | action=getOrder | args=[ABC]
```

---

## Processeur d'annotations à la compilation

Jusqu'ici, nous avons lu les annotations à l'exécution. Mais il est aussi possible de les traiter **à la compilation** grâce à l'API `javax.annotation.processing`.

### Principe

Un processeur d'annotations est invoqué par `javac` pendant la compilation. Il peut :
- Vérifier des contraintes et émettre des erreurs ou warnings
- Générer du code source supplémentaire
- Générer des fichiers de ressources

C'est le mécanisme utilisé par Lombok, MapStruct, Dagger et bien d'autres.

### Créer un processeur simple

Créons un processeur qui vérifie que les classes annotées `@Builder` ont au moins un champ :

```java
@Retention(RetentionPolicy.SOURCE)
@Target(ElementType.TYPE)
public @interface Builder {}
```

```java
import javax.annotation.processing.*;
import javax.lang.model.SourceVersion;
import javax.lang.model.element.*;
import javax.tools.Diagnostic;
import java.util.Set;

@SupportedAnnotationTypes("com.example.Builder")
@SupportedSourceVersion(SourceVersion.RELEASE_17)
public class BuilderProcessor extends AbstractProcessor {

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

### Enregistrer le processeur

Créez le fichier `META-INF/services/javax.annotation.processing.Processor` contenant le nom qualifié du processeur :

```
com.example.BuilderProcessor
```

Ou avec les modules Java, utilisez la directive `provides` dans `module-info.java` :

```java
provides javax.annotation.processing.Processor
    with com.example.BuilderProcessor;
```

---

## Bonnes pratiques

### À faire

- **Toujours spécifier `@Retention` et `@Target`** pour éviter les surprises :

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface MyAnnotation {}
```

- **Utiliser `value()` comme élément principal** quand l'annotation n'a qu'un seul paramètre important.
- **Fournir des valeurs par défaut** quand c'est possible pour simplifier l'utilisation.
- **Documenter avec `@Documented`** pour les annotations d'API publique.
- **Préférer des annotations composées** plutôt que d'empiler les annotations.

### À éviter

- **Ne pas abuser des annotations** : si la logique devient complexe, préférez une approche explicite.
- **Éviter les annotations avec trop d'éléments** : si vous dépassez 5 paramètres, envisagez une classe de configuration.
- **Ne pas utiliser `RetentionPolicy.RUNTIME` sans raison** : cela ajoute des métadonnées au bytecode.
- **Attention à la réflexion** : les appels réflectifs ont un coût de performance, utilisez-les avec discernement.

---

## FAQ

**Quelle est la différence entre `@interface` et `interface` ?**

`@interface` déclare une annotation, `interface` déclare une interface classique. Les annotations ne peuvent pas être instanciées ni implémentées manuellement. Le compilateur génère automatiquement une interface qui étend `java.lang.annotation.Annotation`.

**Peut-on mettre une annotation sur une annotation ?**

Oui, c'est exactement ce que font les méta-annotations (`@Retention`, `@Target`). Utilisez `@Target(ElementType.ANNOTATION_TYPE)` pour cibler les annotations.

**Les annotations ont-elles un impact sur les performances ?**

Les annotations avec `RetentionPolicy.SOURCE` n'ont aucun impact. Celles avec `RUNTIME` ajoutent des métadonnées au bytecode, mais leur impact est négligeable. C'est la **lecture par réflexion** qui peut avoir un coût, surtout si elle est effectuée en boucle.

**Peut-on hériter d'une annotation ?**

Non, les annotations ne supportent pas l'héritage entre elles. Mais `@Inherited` permet aux sous-classes d'hériter des annotations de leur classe parente.

---

## Conclusion

Les annotations personnalisées sont un outil puissant pour enrichir votre code Java avec des métadonnées exploitables. Elles permettent de découpler la logique métier de la logique transversale (validation, logging, sérialisation).

**Points clés à retenir :**

- Déclarez une annotation avec `@interface`
- Configurez-la avec `@Retention` (quand) et `@Target` (où)
- Ajoutez des éléments typés avec des valeurs par défaut
- Lisez-les à l'exécution avec l'API de réflexion (`getAnnotation()`)
- Pour la compilation, créez un `AbstractProcessor`
- Spring, Jakarta EE et la plupart des frameworks Java reposent massivement sur ce mécanisme

Créer ses annotations, c'est comprendre le fonctionnement interne des frameworks que l'on utilise au quotidien.

---

## Pour aller plus loin

- [Documentation Oracle sur les annotations](https://docs.oracle.com/javase/tutorial/java/annotations/)
- [Java Language Specification - Annotations](https://docs.oracle.com/javase/specs/jls/se17/html/jls-9.html#jls-9.6)
- [Guide des Annotation Processors](https://docs.oracle.com/en/java/javase/17/docs/api/java.compiler/javax/annotation/processing/package-summary.html)

## Voir aussi

- [Records en Java : simplifier vos DTOs]({% post_url 2026-01-10-Records-en-Java-simplifier-vos-DTOs %})
- [Les Sealed classes en Java]({% post_url 2026-01-14-Sealed-classes-en-Java %})
- [Optional en Java : éviter les NullPointerException]({% post_url 2026-01-26-Optional-en-Java-eviter-les-NullPointerException %})
- [Pattern matching en Java moderne]({% post_url 2025-10-23-Pattern-matching-en-Java-moderne %})
- [Comment faire des group by en Java]({% post_url 2026-01-11-Comment-faire-des-group-by-en-Java %})
- [Introduction aux collections Java]({% post_url 2020-11-12-Framework-collections-java-intro %})
