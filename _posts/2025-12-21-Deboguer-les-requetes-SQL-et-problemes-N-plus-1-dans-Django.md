---
layout: article
title: "Déboguer les requêtes SQL et problèmes N+1 dans Django"
description: "Afficher les requêtes SQL de Django, repérer les problèmes N+1 (Debug Toolbar, zeal, tests) et les corriger avec select_related, prefetch_related et Prefetch."
tags:
  - python
  - django
  - sql
  - performance
  - orm
  - optimisation
author: Pierre Chopinet
---

Le problème N+1 est l'un des pièges les plus courants avec l'ORM de Django : pour afficher une liste d'objets et leurs relations, Django exécute une requête pour la liste, puis une requête par objet. Avec 100 articles, cela fait 101 requêtes SQL là où une ou deux suffisent. Dans ce tutoriel, nous allons voir comment afficher les requêtes exécutées par Django, repérer les N+1 et les corriger avec `select_related` et `prefetch_related`.
<!--more-->

Dans cet article :
- Afficher les requêtes SQL
- Comprendre le problème N+1
- Corriger avec select_related et prefetch_related
- Précharger une relation filtrée avec Prefetch
- Détecter les N+1 automatiquement
- Tester le nombre de requêtes

Pré-requis : connaître les modèles et les querysets de Django. Les exemples ont été testés avec Django 5.2 sur SQLite.

## Afficher les requêtes SQL

### Avec connection.queries

Quand `DEBUG = True`, Django garde la liste des requêtes exécutées par chaque connexion dans `connection.queries`. Pour un test rapide, on peut la consulter dans le shell (`python manage.py shell`, qui importe automatiquement les modèles depuis Django 5.2). Les modèles utilisés dans cet article sont décrits dans la section suivante.

```python
from django.db import connection

# Exemple : récupérer des articles
articles = list(Article.objects.all())

# Voir les requêtes exécutées
for query in connection.queries:
    print(query['sql'])
    print(f"Time: {query['time']}s\n")

# Nombre total de requêtes
print(f"Total queries: {len(connection.queries)}")
```

Ce qui donne :

```
SELECT "blog_article"."id", "blog_article"."title", "blog_article"."author_id" FROM "blog_article"
Time: 0.001s

Total queries: 1
```

La liste est vidée au début de chaque requête HTTP, et `reset_queries()` la vide à la demande. Avec `DEBUG = False`, elle reste vide. C'est voulu : Django y garderait toutes les requêtes exécutées, ce qui consommerait vite de la mémoire sur un serveur de production.

### Avec le logging

Pour voir passer toutes les requêtes dans la console, on active le logger `django.db.backends` dans `settings.py` :

```python
# settings.py
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
        },
    },
    'loggers': {
        'django.db.backends': {
            'level': 'DEBUG',
            'handlers': ['console'],
        },
    },
}
```

Chaque requête s'affiche alors avec sa durée en secondes et ses paramètres :

```
(0.000) SELECT "blog_article"."id", "blog_article"."title", "blog_article"."author_id" FROM "blog_article"; args=(); alias=default
```

Comme `connection.queries`, ce logger ne fonctionne qu'avec `DEBUG = True` : avec `DEBUG = False`, il reste muet, quel que soit le niveau configuré.

### Avec Django Debug Toolbar

Django Debug Toolbar est l'outil le plus complet pour ça. On l'installe avec pip :

```bash
pip install django-debug-toolbar
```

Puis on le configure :

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'debug_toolbar',
]

MIDDLEWARE = [
    'debug_toolbar.middleware.DebugToolbarMiddleware',
    # ... autres middlewares
]

INTERNAL_IPS = [
    '127.0.0.1',
]
```

```python
# urls.py
from django.urls import path, include
from debug_toolbar.toolbar import debug_toolbar_urls

urlpatterns = [
    # ... vos URLs
] + debug_toolbar_urls()
```

`debug_toolbar_urls()`, disponible depuis la version 4.4.3 de Debug Toolbar, renvoie une liste vide quand `DEBUG` vaut `False` : pas besoin de tester `DEBUG` dans `urls.py`. Le middleware se place le plus tôt possible dans la liste, mais après ceux qui encodent la réponse comme `GZipMiddleware`, sinon Django affiche l'avertissement `debug_toolbar.W003`.

La barre s'affiche sur les pages HTML pour les adresses listées dans `INTERNAL_IPS`. Son panneau SQL donne le nombre de requêtes, la durée de chacune, les requêtes similaires ou dupliquées (le signe d'un N+1) et la pile d'appels qui a déclenché chaque requête.

## Comprendre le problème N+1

Pour la suite, on utilise ces modèles :

```python
# models.py
from django.db import models

class Country(models.Model):
    name = models.CharField(max_length=100)

class Author(models.Model):
    name = models.CharField(max_length=100)
    country = models.ForeignKey(Country, on_delete=models.CASCADE, null=True)

class Tag(models.Model):
    name = models.CharField(max_length=50)

class Article(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
    tags = models.ManyToManyField(Tag)

class Comment(models.Model):
    article = models.ForeignKey(Article, on_delete=models.CASCADE, related_name='comments')
    text = models.TextField()
    published = models.BooleanField(default=False)
```

Le problème N+1 survient quand on boucle sur des objets et qu'on accède à une de leurs relations : Django exécute une requête de plus pour chaque objet. Par exemple :

```python
# views.py
from django.shortcuts import render

from .models import Article

def article_list(request):
    articles = Article.objects.all()  # 1 requête
    for article in articles:
        print(article.author.name)    # N requêtes (1 par article)
    return render(request, 'articles.html', {'articles': articles})
```

Avec 100 articles en base, cette vue exécute 101 requêtes : une pour la liste, puis une par article pour charger son auteur. Voici les trois premières, vues avec `connection.queries` :

```
SELECT "blog_article"."id", "blog_article"."title", "blog_article"."author_id" FROM "blog_article"
SELECT "blog_author"."id", "blog_author"."name", "blog_author"."country_id" FROM "blog_author" WHERE "blog_author"."id" = 2 LIMIT 21
SELECT "blog_author"."id", "blog_author"."name", "blog_author"."country_id" FROM "blog_author" WHERE "blog_author"."id" = 3 LIMIT 21
```

C'est le signe à chercher, dans la Debug Toolbar comme dans les logs : la même requête répétée, avec seulement l'identifiant qui change.

Dans un template, c'est pareil. Chaque accès à `article.author.name` déclenche une requête :

```django
{% raw %}{# templates/articles.html #}
{% for article in articles %}
  <h2>{{ article.title }}</h2>
  <p>Par {{ article.author.name }}</p>  {# 1 requête par article #}
{% endfor %}{% endraw %}
```

## Corriger avec select_related et prefetch_related

Django propose deux méthodes pour charger les relations à l'avance, à appeler au moment où l'on construit le queryset.

### select_related, pour les ForeignKey

Pour une `ForeignKey` ou un `OneToOneField`, `select_related()` ajoute une jointure SQL : l'auteur est récupéré dans la même requête que l'article.

```python
# views.py
def article_list(request):
    # 1 seule requête avec jointure sur Author
    articles = Article.objects.select_related('author').all()
    for article in articles:
        print(article.author.name)  # Pas de requête supplémentaire
    return render(request, 'articles.html', {'articles': articles})
```

La requête générée :

```sql
SELECT "blog_article"."id", "blog_article"."title", "blog_article"."author_id", "blog_author"."id", "blog_author"."name", "blog_author"."country_id" FROM "blog_article" INNER JOIN "blog_author" ON ("blog_article"."author_id" = "blog_author"."id")
```

Une seule requête au lieu de 101.

### prefetch_related, pour les ManyToMany et les relations inverses

Pour une relation qui renvoie plusieurs objets (`ManyToManyField`, ou `ForeignKey` vue de l'autre côté comme les commentaires d'un article), une jointure multiplierait les lignes du résultat. `select_related` ne le fait donc pas, et Django lève une erreur si on essaie :

```
FieldError: Invalid field name(s) given in select_related: 'tags'. Choices are: author
```

Pour ces relations, on utilise `prefetch_related()` : Django exécute une requête séparée pour la relation, puis rattache les résultats aux objets en Python. La version naïve :

```python
articles = Article.objects.all()
for article in articles:
    for tag in article.tags.all():  # N requêtes
        print(tag.name)
```

Et la version optimisée :

```python
articles = Article.objects.prefetch_related('tags').all()
for article in articles:
    for tag in article.tags.all():  # Pas de requête supplémentaire
        print(tag.name)
```

On passe de 101 à 2 requêtes. La seconde récupère d'un coup les tags de tous les articles (la liste des identifiants est raccourcie ici, elle va jusqu'à 100) :

```sql
SELECT "blog_article"."id", "blog_article"."title", "blog_article"."author_id" FROM "blog_article"
SELECT ("blog_article_tags"."article_id") AS "_prefetch_related_val_article_id", "blog_tag"."id", "blog_tag"."name" FROM "blog_tag" INNER JOIN "blog_article_tags" ON ("blog_tag"."id" = "blog_article_tags"."tag_id") WHERE "blog_article_tags"."article_id" IN (1, 2, 3, ..., 100)
```

### Plusieurs niveaux de relations

On peut suivre plusieurs relations avec la syntaxe `__` :

```python
# Article -> Author -> Country
articles = Article.objects.select_related('author', 'author__country').all()
```

C'est toujours une seule requête, avec une seconde jointure. Comme le pays d'un auteur est facultatif (`null=True`), Django utilise ici un `LEFT OUTER JOIN` pour ne pas perdre les articles dont l'auteur n'a pas de pays.

Et on peut combiner les deux méthodes :

```python
# Article a une ForeignKey vers Author, et un ManyToMany vers Tag
articles = Article.objects.select_related('author').prefetch_related('tags').all()
```

Deux requêtes en tout : la jointure avec les auteurs, puis les tags.

## Précharger une relation filtrée avec Prefetch

Pour n'afficher que les commentaires publiés de chaque article, le réflexe est de filtrer dans la boucle. Mais tout appel qui change la requête (`filter()`, `order_by()`...) ignore les objets préchargés et repart en base :

```python
articles = Article.objects.prefetch_related('comments').all()
for article in articles:
    # filter() ignore les commentaires préchargés : 1 requête par article
    for comment in article.comments.filter(published=True):
        print(comment.text)
```

Sur nos 100 articles, cela fait 102 requêtes : les deux du préchargement, devenu inutile, plus une par article. Pour précharger directement une relation filtrée, on passe par un objet `Prefetch` :

```python
from django.db.models import Prefetch

articles = Article.objects.prefetch_related(
    Prefetch(
        'comments',
        queryset=Comment.objects.filter(published=True),
        to_attr='published_comments'
    )
).all()

for article in articles:
    for comment in article.published_comments:  # préchargé et filtré
        print(comment.text)
```

Deux requêtes en tout. `to_attr` range le résultat dans une simple liste, `article.published_comments`, au lieu de remplir le cache de `article.comments`. La documentation le recommande dès qu'on filtre : sinon, `article.comments.all()` ne renverrait que les commentaires publiés, sans que rien ne l'indique dans le code.

Filtrer le queryset principal ne pose pas de problème, en revanche : avec `Article.objects.prefetch_related('comments').filter(...)`, le préchargement est conservé et porte sur les articles filtrés.

## Détecter les N+1 automatiquement

### django-querycount

Ce middleware affiche dans la console, pour chaque requête HTTP, le nombre de requêtes SQL et les doublons. Il s'appuie sur `connection.queries` et ne fonctionne donc qu'avec `DEBUG = True`. Les exemples ci-dessous utilisent la version 0.8.3.

```bash
pip install django-querycount
```

```python
# settings.py
MIDDLEWARE = [
    'querycount.middleware.QueryCountMiddleware',
    # ... autres middlewares
]

QUERYCOUNT = {
    'DISPLAY_DUPLICATES': 1,  # nombre de requêtes dupliquées à afficher
    'RESPONSE_HEADER': 'X-DjangoQueryCount-Count',
}
```

Sur une vue qui affiche les 100 articles et leur auteur sans `select_related`, la console de `runserver` affiche :

```
http://127.0.0.1:8000/articles/
|------|-----------|----------|----------|----------|------------|
| Type | Database  |   Reads  |  Writes  |  Totals  | Duplicates |
|------|-----------|----------|----------|----------|------------|
| RESP |  default  |   101    |    0     |   101    |    100     |
|------|-----------|----------|----------|----------|------------|
Total queries: 101 in 0.0372s


Executed 100 time(s).
SELECT "blog_author"."id", "blog_author"."name",
"blog_author"."country_id" FROM "blog_author" WHERE "blog_author"."id"
= #number# LIMIT 21
```

`DISPLAY_DUPLICATES` attend un nombre : celui des requêtes dupliquées à afficher, des plus fréquentes aux moins fréquentes. Les identifiants y sont remplacés par `#number#`, ce qui fait ressortir la requête répétée. Le total est aussi renvoyé dans l'en-tête de réponse `X-DjangoQueryCount-Count`.

### django-zeal

On voit souvent `nplusone` recommandé pour détecter les N+1, mais il n'a plus eu de nouvelle version depuis la 1.0.0 de mai 2018. Il repère encore le cas simple de cet article avec Django 5.2. Pour un projet actuel, on lui préfère tout de même `django-zeal`, qui s'en inspire et reste maintenu.

```bash
pip install django-zeal
```

```python
# settings.py
if DEBUG:
    INSTALLED_APPS.append("zeal")
    MIDDLEWARE.append("zeal.middleware.zeal_middleware")

# Par défaut, zeal lève une exception dès qu'un N+1 est détecté.
# Passez ZEAL_RAISE = False pour n'émettre que des warnings.
ZEAL_RAISE = True
```

Sur la vue naïve présentée plus haut, zeal lève cette erreur (chemin du projet raccourci) :

```
NPlusOneError: N+1 detected on blog.Article.author at .../blog/views.py:9 in article_list
```

Le message donne la relation en cause et la ligne de code qui a déclenché les requêtes. D'après sa documentation, zeal ajoute de l'ordre de 3 à 5 % de temps d'exécution : il est fait pour le développement et les tests, pas pour la production.

## Tester le nombre de requêtes

Pour qu'un N+1 corrigé ne revienne pas, le plus sûr est d'ajouter un test. `TestCase` fournit `assertNumQueries`, qui échoue si le bloc n'exécute pas exactement le nombre de requêtes attendu :

```python
from django.test import TestCase

from .models import Article, Author

class ArticleViewTest(TestCase):
    @classmethod
    def setUpTestData(cls):
        for i in range(10):
            author = Author.objects.create(name=f"Auteur {i}")
            Article.objects.create(title=f"Article {i}", author=author)

    def test_article_list_queries(self):
        with self.assertNumQueries(1):
            self.client.get('/articles/')
```

Avec la vue qui utilise `select_related`, le test passe. Avec la version naïve, il échoue et liste les requêtes exécutées :

```
AssertionError: 11 != 1 : 11 queries executed, 1 expected
Captured queries were:
1. SELECT "blog_article"."id", "blog_article"."title", "blog_article"."author_id" FROM "blog_article"
2. SELECT "blog_author"."id", "blog_author"."name", "blog_author"."country_id" FROM "blog_author" WHERE "blog_author"."id" = 1 LIMIT 21
3. SELECT "blog_author"."id", "blog_author"."name", "blog_author"."country_id" FROM "blog_author" WHERE "blog_author"."id" = 2 LIMIT 21
4. SELECT "blog_author"."id", "blog_author"."name", "blog_author"."country_id" FROM "blog_author" WHERE "blog_author"."id" = 3 LIMIT 21
5. SELECT "blog_author"."id", "blog_author"."name", "blog_author"."country_id" FROM "blog_author" WHERE "blog_author"."id" = 4 LIMIT 21
6. SELECT "blog_author"."id", "blog_author"."name", "blog_author"."country_id" FROM "blog_author" WHERE "blog_author"."id" = 5 LIMIT 21
7. SELECT "blog_author"."id", "blog_author"."name", "blog_author"."country_id" FROM "blog_author" WHERE "blog_author"."id" = 6 LIMIT 21
8. SELECT "blog_author"."id", "blog_author"."name", "blog_author"."country_id" FROM "blog_author" WHERE "blog_author"."id" = 7 LIMIT 21
9. SELECT "blog_author"."id", "blog_author"."name", "blog_author"."country_id" FROM "blog_author" WHERE "blog_author"."id" = 8 LIMIT 21
10. SELECT "blog_author"."id", "blog_author"."name", "blog_author"."country_id" FROM "blog_author" WHERE "blog_author"."id" = 9 LIMIT 21
11. SELECT "blog_author"."id", "blog_author"."name", "blog_author"."country_id" FROM "blog_author" WHERE "blog_author"."id" = 10 LIMIT 21
```

Pour vérifier un maximum plutôt qu'un nombre exact, `CaptureQueriesContext` enregistre les requêtes d'un bloc :

```python
from django.db import connection
from django.test.utils import CaptureQueriesContext

class ArticleViewTest(TestCase):
    def test_article_list_max_queries(self):
        with CaptureQueriesContext(connection) as context:
            self.client.get('/articles/')
        self.assertLessEqual(len(context.captured_queries), 3)
```

Les tests de Django tournent toujours avec `DEBUG = False`, mais ces deux outils activent eux-mêmes l'enregistrement des requêtes : ils fonctionnent sans rien configurer.

## Voir aussi

- [Comment ajouter du cache à une application Django]({% post_url 2025-11-01-Comment-ajouter-du-cache-a-une-application-Django %})
- [Accélérer Django avec la compression HTTP]({% post_url 2025-12-13-Accelerer-Django-avec-la-compression-GZip %})
- [Documentation de Django sur l'optimisation des accès à la base](https://docs.djangoproject.com/en/5.2/topics/db/optimization/)
- [Documentation de Django Debug Toolbar](https://django-debug-toolbar.readthedocs.io/)
- [django-querycount sur PyPI](https://pypi.org/project/django-querycount/)
- [django-zeal sur PyPI](https://pypi.org/project/django-zeal/)
