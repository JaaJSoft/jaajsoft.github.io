---
layout: article
title: "Déboguer les requêtes SQL et problèmes N+1 dans Django"
author: Pierre Chopinet
tags:
  - python
  - django
  - sql
  - performance
  - orm
  - optimisation
---

Le problème N+1 est l'un des pièges les plus courants en Django : pour afficher une liste d'objets avec leurs relations, l'ORM exécute 1 requête pour la liste + N requêtes (une par objet). Résultat : des centaines de requêtes SQL qui plombent les performances. Dans ce guide, on voit comment visualiser toutes les requêtes SQL de votre app Django, détecter les N+1, et les corriger avec `select_related` et `prefetch_related`.
<!--more-->

Objectifs:
- Afficher et analyser toutes les requêtes SQL exécutées par Django
- Comprendre le problème N+1 et savoir le détecter
- Optimiser avec `select_related()` (jointures) et `prefetch_related()` (requêtes séparées)
- Utiliser des outils comme Django Debug Toolbar, `django-querycount`, et le logging SQL
- Éviter les pièges courants et adopter les bonnes pratiques

Prérequis :
- Django 4.2+ (ou 5.x)
- Notions de modèles et querysets

Avant de résoudre les problèmes, commençons par les rendre visibles : voyons comment afficher les requêtes SQL.

---

## 1) Afficher les requêtes SQL dans Django

Django propose plusieurs méthodes pour voir les requêtes SQL exécutées par l'ORM.

### a) Via `connection.queries`

Utile pour un debug rapide dans le shell ou une vue de dev.

```python
# settings.py
DEBUG = True  # requis pour que connection.queries fonctionne

# shell Django ou view
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

> **Attention** : `connection.queries` ne fonctionne qu'avec `DEBUG = True` et peut consommer beaucoup de mémoire en production.

### b) Logging SQL

Activez le logging des requêtes SQL dans `settings.py` :

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

Toutes les requêtes SQL s'afficheront dans la console :

```
(0.001) SELECT "blog_article"."id", "blog_article"."title", ... FROM "blog_article"; args=()
```

### c) Django Debug Toolbar

L'outil le plus complet pour visualiser les requêtes, temps d'exécution, requêtes dupliquées, etc.

Installation :

```bash
pip install django-debug-toolbar
```

Configuration :

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
from django.conf import settings

urlpatterns = [
    # ... vos URLs
]

if settings.DEBUG:
    import debug_toolbar
    urlpatterns = [
        path('__debug__/', include(debug_toolbar.urls)),
    ] + urlpatterns
```

Une fois installé, un panneau latéral apparaît dans votre navigateur avec :
- Le nombre total de requêtes
- Le temps d'exécution de chaque requête
- Les requêtes dupliquées (= problèmes N+1)
- Les traces de code qui ont déclenché chaque requête

Maintenant que vous voyez les requêtes, passons au problème le plus courant : le N+1.

---

## 2) Comprendre le problème N+1

Le problème N+1 survient quand on boucle sur des objets et qu'on accède à leurs relations : Django exécute une requête supplémentaire pour chaque objet.

### Exemple classique

Modèles :

```python
# models.py
from django.db import models

class Author(models.Model):
    name = models.CharField(max_length=100)

class Article(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
```

Vue naïve (problème N+1) :

```python
# views.py
def article_list(request):
    articles = Article.objects.all()  # 1 requête
    for article in articles:
        print(article.author.name)    # N requêtes (1 par article)
    return render(request, 'articles.html', {'articles': articles})
```

**Résultat** : Si vous avez 100 articles, Django exécute :
- 1 requête pour récupérer les 100 articles
- 100 requêtes pour récupérer l'auteur de chaque article
- **Total : 101 requêtes**

Dans le template, c'est pareil :

```django
{# templates/articles.html #}
{% for article in articles %}
  <h2>{{ article.title }}</h2>
  <p>Par {{ article.author.name }}</p>  {# 1 requête par article #}
{% endfor %}
```

### Comment détecter un N+1 ?

- Django Debug Toolbar : section « SQL » → regardez si vous voyez des requêtes répétées (ex : `SELECT ... FROM author WHERE id = ?` avec différents IDs)
- `connection.queries` : comptez le nombre de requêtes et cherchez des patterns répétitifs
- Package `django-querycount` (voir section 5)

Passons maintenant aux solutions pour éliminer ces requêtes en trop.

---

## 3) Corriger le N+1 : select_related() et prefetch_related()

Django offre deux outils puissants pour charger les relations en une seule fois.

### a) select_related() : jointures SQL (ForeignKey, OneToOne)

Utilisez `select_related()` pour les relations **un-à-un** ou **plusieurs-à-un** (ForeignKey, OneToOneField). Django effectue une jointure SQL et récupère tout en une seule requête.

Version optimisée :

```python
# views.py
def article_list(request):
    # 1 seule requête avec jointure sur Author
    articles = Article.objects.select_related('author').all()
    for article in articles:
        print(article.author.name)  # Pas de requête supplémentaire !
    return render(request, 'articles.html', {'articles': articles})
```

**SQL généré** :
```sql
SELECT article.*, author.*
FROM article
INNER JOIN author ON article.author_id = author.id
```

**Résultat** : 1 seule requête au lieu de 101

### b) prefetch_related() : requêtes séparées (ManyToMany, reverse ForeignKey)

Pour les relations **un-à-plusieurs** ou **plusieurs-à-plusieurs**, utilisez `prefetch_related()`. Django effectue 2 requêtes séparées mais optimisées (une pour les objets principaux, une pour les relations), puis les relie en Python.

Modèles :

```python
# models.py
class Article(models.Model):
    title = models.CharField(max_length=200)
    tags = models.ManyToManyField('Tag')

class Tag(models.Model):
    name = models.CharField(max_length=50)
```

Vue naïve (N+1) :

```python
articles = Article.objects.all()
for article in articles:
    for tag in article.tags.all():  # N requêtes
        print(tag.name)
```

Version optimisée :

```python
articles = Article.objects.prefetch_related('tags').all()
for article in articles:
    for tag in article.tags.all():  # Pas de requête supplémentaire
        print(tag.name)
```

**SQL généré** :
```sql
-- Requête 1 : articles
SELECT * FROM article;
-- Requête 2 : tags (avec IDs des articles)
SELECT tag.*, article_tags.article_id
FROM tag
INNER JOIN article_tags ON tag.id = article_tags.tag_id
WHERE article_tags.article_id IN (1, 2, 3, ...);
```

**Résultat** : 2 requêtes au total (au lieu de N+1).

### c) Chaîner plusieurs relations

Vous pouvez optimiser plusieurs niveaux de relations :

```python
# Modèles : Article -> Author -> Country
articles = Article.objects.select_related('author', 'author__country').all()
```

Ou combiner les deux :

```python
# Article a un ForeignKey vers Author, et un ManyToMany vers Tag
articles = Article.objects.select_related('author').prefetch_related('tags').all()
```

---

## 4) Cas avancés : Prefetch() et filtrage

Parfois, vous voulez précharger une relation mais avec un filtrage ou un tri personnalisé.

### Exemple : récupérer uniquement les commentaires publiés

Modèles :

```python
class Article(models.Model):
    title = models.CharField(max_length=200)

class Comment(models.Model):
    article = models.ForeignKey(Article, on_delete=models.CASCADE, related_name='comments')
    text = models.TextField()
    published = models.BooleanField(default=False)
```

Sans optimisation :

```python
articles = Article.objects.all()
for article in articles:
    # N+1 si on filtre dans la boucle
    published_comments = article.comments.filter(published=True)
```

Avec `Prefetch()` :

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

**Avantages** :
- 2 requêtes au total (1 pour les articles, 1 pour les commentaires publiés)
- `to_attr` crée un attribut custom pour éviter de polluer le cache du relation manager

---

## 5) Outils pour détecter et surveiller les N+1

### a) django-querycount

Ce package affiche automatiquement le nombre de requêtes pour chaque requête HTTP dans la console.

Installation :

```bash
pip install django-querycount
```

Configuration :

```python
# settings.py
MIDDLEWARE = [
    'querycount.middleware.QueryCountMiddleware',
    # ... autres middlewares
]

QUERYCOUNT = {
    'DISPLAY_DUPLICATES': True,  # affiche les requêtes dupliquées
    'RESPONSE_HEADER': 'X-DjangoQueryCount-Count',
}
```

Résultat dans la console :

```
[SQL] GET /articles/ : 101 queries (99 duplicates)
```

### b) nplusone (détection automatique)

Ce package détecte les N+1 et génère des warnings.

```bash
pip install nplusone
```

```python
# settings.py
INSTALLED_APPS = [
    'nplusone.ext.django',
]

NPLUSONE_RAISE = True  # lève une exception si N+1 détecté (dev uniquement)
```

---

## 6) Bonnes pratiques et pièges à éviter

### Bonnes pratiques

- **Utilisez `select_related()` et `prefetch_related()` dès la définition du queryset**, pas dans la boucle.
- **Profilez** : activez Debug Toolbar ou logging SQL systématiquement en dev.
- **Testez** : écrivez des tests d'assertion sur le nombre de requêtes :

```python
from django.test import TestCase
from django.test.utils import override_settings

class ArticleViewTest(TestCase):
    def test_article_list_queries(self):
        # Créez des données de test
        from django.db import connection
        from django.test.utils import CaptureQueriesContext

        with CaptureQueriesContext(connection) as context:
            response = self.client.get('/articles/')

        # Vérifiez qu'on a bien optimisé
        self.assertLessEqual(len(context.captured_queries), 3)
```

### Pièges courants

- **Ne pas appliquer `select_related()` après avoir itéré** : ça ne change rien, le queryset est déjà évalué.
- **Filtrer après `prefetch_related()`** : si vous faites `.filter()` après, le prefetch est ignoré. Utilisez `Prefetch()` pour filtrer.
- **Utiliser `select_related()` sur un ManyToMany** : ça ne marche pas, utilisez `prefetch_related()`.
- **Oublier les relations imbriquées** : pensez à `select_related('author__country')`.

---

## Cheatsheet

- Afficher les requêtes :
  ```python
  from django.db import connection
  print(len(connection.queries))
  ```
- Logging SQL :
  ```python
  # settings.py
  LOGGING = {'loggers': {'django.db.backends': {'level': 'DEBUG'}}}
  ```
- ForeignKey/OneToOne → `select_related()` :
  ```python
  Article.objects.select_related('author', 'author__country')
  ```
- ManyToMany/reverse FK → `prefetch_related()` :
  ```python
  Article.objects.prefetch_related('tags')
  ```
- Préchargement personnalisé :
  ```python
  from django.db.models import Prefetch
  Article.objects.prefetch_related(
      Prefetch('comments', queryset=Comment.objects.filter(published=True))
  )
  ```
- Django Debug Toolbar : indispensable en dev.
- `django-querycount` : affiche le nombre de requêtes par vue.

---

## Conclusion

Visualiser et déboguer les requêtes SQL est essentiel pour optimiser une app Django. Le problème N+1 est insidieux mais facile à résoudre avec `select_related()` et `prefetch_related()`. Activez Django Debug Toolbar en dev, profilez vos vues critiques, et testez le nombre de requêtes. Vos utilisateurs (et votre serveur) vous remercieront.

---

## Pour aller plus loin

- [Comment ajouter du cache à une application Django]({% post_url 2025-11-01-Comment-ajouter-du-cache-a-une-application-Django %})
- [Accélérer Django avec la compression GZip]({% post_url 2025-12-13-Accelerer-Django-avec-la-compression-GZip %})
- [Documentation officielle Django (optimisation de base de données)](https://docs.djangoproject.com/en/stable/topics/db/optimization/)
