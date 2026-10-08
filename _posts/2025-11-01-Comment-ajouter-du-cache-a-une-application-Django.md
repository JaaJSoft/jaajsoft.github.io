---
layout: article
title: "Comment ajouter du cache à une application Django"
description: "Ajouter du cache à une application Django : backends, cache par vue, par site et de fragments, API bas niveau, invalidation et cache avec Django REST Framework."
tags:
  - python
  - django
  - cache
  - performance
  - redis
  - memcached
author: Pierre Chopinet
---

Une page qui refait les mêmes requêtes SQL à chaque affichage fait travailler le serveur pour rien. Le framework de cache de Django permet de garder le résultat à plusieurs niveaux : la page entière, un morceau de template ou une simple valeur calculée. Dans ce tutoriel, nous allons configurer un backend de cache, utiliser chacun de ces niveaux, puis voir comment invalider le cache proprement.
<!--more-->

Dans cet article :
- Choisir un backend de cache
- Mettre une vue en cache
- Le cache de tout le site
- Mettre en cache un fragment de template
- L'API bas niveau
- Invalider le cache
- Contenu par utilisateur, langues et pagination
- Django REST Framework
- Vérifier que le cache sert

Pré-requis : connaître les vues et les templates Django. Les exemples ont été testés avec Django 5.2.

## Choisir un backend de cache

Le backend se configure dans le réglage `CACHES` (sans configuration, Django utilise le cache en mémoire locale). `TIMEOUT` y fixe la durée de vie par défaut en secondes, 300 si on ne le précise pas, et `KEY_PREFIX` est ajouté devant toutes les clés pour éviter les collisions quand plusieurs projets partagent le même serveur de cache.

### Mémoire locale

Aucune dépendance, et c'est très rapide :

```python
# settings.py
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.locmem.LocMemCache",
        "LOCATION": "unique-locmem",  # nom du cache en mémoire
        "TIMEOUT": 300,                # secondes (None = jamais expirer)
        "KEY_PREFIX": "myapp",
    }
}
```

Par contre, chaque processus a son propre cache : avec 4 workers Gunicorn, vous avez 4 caches indépendants, et rien n'est partagé entre deux conteneurs. C'est très bien pour le développement, à éviter en production.

### Fichier

Le cache survit aux redémarrages, au prix d'accès disque :

```python
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.filebased.FileBasedCache",
        "LOCATION": BASE_DIR / "django_cache",  # dossier créé au besoin
        "TIMEOUT": 600,
    }
}
```

L'utilisateur du serveur doit pouvoir écrire dans ce dossier, qui ne doit pas se trouver dans `MEDIA_ROOT` ou `STATIC_ROOT` : les valeurs sont sérialisées avec pickle, et la documentation de Django prévient qu'un attaquant qui accède à ces fichiers pourrait aller jusqu'à exécuter du code.

### Base de données

Pas de service en plus à installer, mais chaque accès au cache devient une requête SQL :

```python
# 1) Créer la table
# python manage.py createcachetable my_cache_table

CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.db.DatabaseCache",
        "LOCATION": "my_cache_table",
        "TIMEOUT": 600,
    }
}
```

### Memcached

Memcached est un serveur de cache en mémoire, partagé par tous les processus qui s'y connectent. Django le pilote avec la bibliothèque `pymemcache` (ou `pylibmc`) :

```python
CACHES = {
    "default": {
        # Requiert 'pymemcache' (ou 'pylibmc')
        "BACKEND": "django.core.cache.backends.memcached.PyMemcacheCache",
        "LOCATION": ["127.0.0.1:11211"],
        "TIMEOUT": 300,
        "KEY_PREFIX": "myapp",
    }
}
```

Attention à la taille des valeurs : par défaut, Memcached refuse tout ce qui dépasse 1 Mo, et pymemcache lève alors `MemcacheServerError: b'object too large for cache'`.

### Redis

Redis est partagé entre processus et serveurs, gère l'expiration des clés et propose des opérations en plus (verrous, compteurs...). C'est le backend que je vous conseille pour la plupart des applications. Django en fournit un depuis la version 4.0, basé sur la bibliothèque `redis` (`pip install redis`) :

```python
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.redis.RedisCache",
        "LOCATION": "redis://127.0.0.1:6379",
    }
}
```

Le paquet `django-redis` reste utile pour ses fonctions en plus : suppression par motif, compression des valeurs, ou encore l'option `IGNORE_EXCEPTIONS` :

```python
# pip install django-redis
CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": "redis://127.0.0.1:6379/1",
        "OPTIONS": {
            "CLIENT_CLASS": "django_redis.client.DefaultClient",
            # Compression optionnelle
            # "COMPRESSOR": "django_redis.compressors.zlib.ZlibCompressor",
            # Ne pas planter si Redis est indisponible
            "IGNORE_EXCEPTIONS": True,
        },
        "TIMEOUT": 300,
        "KEY_PREFIX": "myapp",
    }
}
```

Sans cette option, et avec le backend natif, un Redis injoignable fait lever `redis.exceptions.ConnectionError` à chaque accès au cache, et la page part en erreur 500. Avec `IGNORE_EXCEPTIONS`, le cache se comporte comme un cache vide : `get` renvoie `None` et l'application continue de répondre, plus lentement.

## Mettre une vue en cache

Le cache par vue est le plus simple à mettre en place, pour des pages identiques pour tout le monde (une page d'accueil publique par exemple) :

```python
# views.py
from django.views.generic import TemplateView
from django.utils.decorators import method_decorator
from django.views.decorators.cache import cache_page, cache_control

@method_decorator(cache_page(60 * 15), name="dispatch")  # 15 min
@method_decorator(cache_control(public=True), name="dispatch")
class HomeView(TemplateView):
    template_name = "home.html"
```

```python
# urls.py
from django.urls import path
from .views import HomeView

urlpatterns = [
    path("", HomeView.as_view(), name="home"),
]
```

`cache_page` garde la réponse complète pendant 15 minutes. La clé est construite à partir de l'URL complète, query string comprise : `/articles/?page=2` et `/articles/?page=3` sont deux entrées différentes. Le décorateur ajoute aussi les en-têtes `Cache-Control` et `Expires`, et `cache_control(public=True)` autorise les caches intermédiaires (CDN, proxy) à garder la page. La réponse contient alors :

```
Cache-Control: public, max-age=900
```

Si le contenu dépend d'un en-tête de la requête, `vary_on_headers` l'ajoute à l'en-tête `Vary` et Django garde une version de la page par valeur de cet en-tête :

```python
from django.views.decorators.vary import vary_on_headers

@method_decorator(vary_on_headers("User-Agent"), name="dispatch")
class HomeView(...):
    ...
```

Pour la langue, rien à faire : avec `USE_I18N = True` (la valeur d'un nouveau projet), la langue active fait déjà partie de la clé, tout comme le fuseau horaire avec `USE_TZ = True`.

Attention, une page qui dépend de l'utilisateur connecté ne doit pas passer telle quelle par `cache_page` : on risque de servir la page d'un utilisateur à un autre (voir plus bas).

## Le cache de tout le site

Pour mettre en cache toutes les pages d'un site public, Django fournit deux middlewares :

```python
# settings.py (ordre important)
MIDDLEWARE = [
    "django.middleware.cache.UpdateCacheMiddleware",    # 1er
    # ... vos middlewares habituels ...
    "django.middleware.cache.FetchFromCacheMiddleware", # dernier
]

CACHE_MIDDLEWARE_SECONDS = 600
CACHE_MIDDLEWARE_KEY_PREFIX = "mysite"
```

L'ordre n'est pas une faute de frappe. `UpdateCacheMiddleware` enregistre la réponse, et les middlewares traitent les réponses de bas en haut : en tête de liste, il passe en dernier, après ceux qui modifient l'en-tête `Vary` (sessions, compression, langue). `FetchFromCacheMiddleware` cherche la page pendant le traitement de la requête, qui se fait de haut en bas : il doit lui aussi passer après eux.

Seules les réponses 200 aux requêtes `GET` et `HEAD` sont mises en cache, une par URL et par query string. Pour exclure une vue :

```python
# views.py
from django.views.decorators.cache import never_cache

@never_cache
def admin_dashboard(request):
    ...
```

`never_cache` ajoute `Cache-Control: max-age=0, no-cache, no-store, must-revalidate, private`, et le middleware ne garde pas la réponse. Comme le cache par vue, le cache par site ne convient qu'aux pages identiques pour tous.

## Mettre en cache un fragment de template

Quand seule une partie de la page coûte cher (une barre latérale, un menu calculé), on met en cache ce fragment avec la balise `cache` :

{% raw %}
```django
{# template.html #}
{% load cache %}

<main>
  <h1>{{ page_title }}</h1>

  {% cache 600 sidebar user.pk %}
    {# Ce bloc est mis en cache 10 min, clef inclut l'ID user #}
    {% include "_sidebar.html" %}
  {% endcache %}

  <section>
    ... contenu principal ...
  </section>
</main>
```
{% endraw %}

La balise prend une durée en secondes, un nom de fragment, puis autant d'arguments que nécessaire pour distinguer les versions : ici une par utilisateur, grâce à `user.pk`. Chaque combinaison d'arguments crée une entrée dans le cache, évitez donc les arguments qui prennent un très grand nombre de valeurs différentes.

## L'API bas niveau

Pour mettre en cache le résultat d'un calcul plutôt qu'un morceau de HTML, on utilise directement l'objet `cache` :

```python
# services.py
from django.core.cache import cache

KEY = "stats:homepage"

def get_home_stats():
    def _compute():
        # Simule un calcul coûteux
        return {"articles": 42, "users": 1337}

    # get_or_set calcule et stocke si absent
    return cache.get_or_set(KEY, _compute, timeout=300)
```

`get_or_set` accepte une fonction en valeur par défaut : elle n'est appelée que si la clé est absente du cache.

Les autres opérations courantes, avec ce qu'elles renvoient :

```python
cache.set("foo", {"x": 1}, timeout=60)
cache.get("foo")                     # {'x': 1}
cache.add("foo", 2, timeout=60)      # False : add n'écrase pas une clé existante
cache.set("counter", 10)
cache.incr("counter", delta=1)       # 11 (ValueError si la clé n'existe pas)
cache.decr("counter", delta=2)       # 9
cache.delete("foo")                  # True (False si la clé n'existait pas)
cache.delete_many(["k1", "k2"])
cache.touch("counter", timeout=120)  # True : nouvelle durée de vie
```

`touch` renvoie `False` si la clé n'existe pas. `incr` et `decr` ne sont atomiques que si le backend sait le faire nativement (Memcached par exemple) ; sinon Django lit la valeur puis réécrit le résultat.

Les valeurs sont sérialisées avec pickle : on peut stocker tout objet Python sérialisable, des dictionnaires comme des listes d'objets de modèle. Avec django-redis, l'option `COMPRESSOR` vue plus haut réduit la place qu'elles occupent.

## Invalider le cache

Le plus simple reste une durée de vie réaliste, 5 à 15 minutes pour du contenu éditorial par exemple. Quand une ressource change, on supprime sa clé :

```python
from django.core.cache import cache

def invalidate_article(article_id: int):
    cache.delete(f"article:{article_id}")
```

Avec django-redis, on peut aussi supprimer toutes les clés qui suivent un motif :

```python
from django.core.cache import cache

cache.delete_pattern("user:*")  # renvoie le nombre de clés supprimées
```

Attention à ne pas passer directement par le client Redis (`get_redis_connection("default").scan_iter("user:*")`) : Django range les clés sous la forme `préfixe:version:clé`, ici `myapp:1:user:42`, et ce motif ne trouve rien. `delete_pattern` ajoute lui-même le préfixe et la version.

Pour un fragment de template, `make_template_fragment_key` retrouve la clé à partir du nom du fragment et de ses arguments :

```python
from django.core.cache import cache
from django.core.cache.utils import make_template_fragment_key

# fragment "sidebar" de l'utilisateur 42
cache.delete(make_template_fragment_key("sidebar", [42]))
```

Lors d'un déploiement qui change le format des données en cache, le plus simple est de changer de version. Le réglage `VERSION` s'applique à toutes les clés, et les valeurs écrites avec l'ancienne version sont ignorées :

```python
# settings.py
CACHES = {
    "default": {
        # ...
        "VERSION": 2,
    }
}
```

Pour une seule clé, on peut passer `version=` à `get` et `set`, ou utiliser `incr_version` :

```python
cache.set("home", "ancien format")
cache.incr_version("home")    # 2
cache.get("home")             # None
cache.get("home", version=2)  # 'ancien format'
```

Enfin, les signaux permettent d'invalider le cache dès qu'un objet est enregistré ou supprimé :

```python
# signals.py
from django.db.models.signals import post_save, post_delete
from django.dispatch import receiver
from .models import Article
from django.core.cache import cache

@receiver([post_save, post_delete], sender=Article)
def invalidate_article_cache(sender, instance, **kwargs):
    cache.delete(f"article:{instance.pk}")
```

Le receiver n'est enregistré que si le module `signals` est importé. Comme le recommande la documentation, on l'importe dans la méthode `ready()` de la configuration de l'application :

```python
# apps.py
from django.apps import AppConfig

class BlogConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "blog"

    def ready(self):
        from . import signals  # noqa: F401
```

## Contenu par utilisateur, langues et pagination

### Par utilisateur

Pour une page qui dépend de l'utilisateur, évitez de mettre en cache la page complète. Cachez plutôt des fragments ou des données, avec l'identifiant de l'utilisateur dans la clé :

```python
from django.core.cache import cache

def get_user_dashboard(user):
    key = f"dash:{user.pk}"
    return cache.get_or_set(key, lambda: compute_dashboard(user), 300)
```

Si vous tenez à mettre la page entière en cache, `vary_on_cookie` crée une version par valeur de l'en-tête `Cookie`, donc en pratique une par session :

```python
from django.views.decorators.cache import cache_page
from django.views.decorators.vary import vary_on_cookie

@cache_page(300)
@vary_on_cookie
def public_but_personalized(request):
    ...
```

L'ordre des décorateurs compte : `cache_page` doit être au-dessus, pour voir l'en-tête `Vary: Cookie` posé par `vary_on_cookie` et inclure le cookie dans la clé. Dans l'ordre inverse, la clé ignore le cookie : sur une vue qui affiche un nom lu dans un cookie, Bob reçoit alors la page mise en cache pour Alice.

Même dans le bon ordre, une entrée par session peut vite faire beaucoup de clés, les fragments ciblés restent préférables. Et de façon générale, ne mettez pas de données sensibles (jetons, informations personnelles) dans un cache partagé.

### Langues

Comme vu plus haut, les caches par vue et par site tiennent déjà compte de la langue active. Avec l'API bas niveau, ajoutez le code de la langue dans la clé :

```python
from django.utils.translation import get_language
key = f"home:{get_language()}"
```

### Pagination et tri

Les caches par vue et par site incluent la query string dans la clé. Avec l'API bas niveau, c'est à vous d'y mettre les paramètres :

```python
page = request.GET.get("page", "1")
sort = request.GET.get("sort", "-date")
key = f"list:{page}:{sort}"
```

Attention, ces valeurs viennent de l'utilisateur. Avec Memcached, une clé qui contient un espace ou dépasse 250 caractères lève `InvalidCacheKey` (les autres backends se contentent d'un avertissement `CacheKeyWarning`). Validez les paramètres avant de construire la clé, ou hachez-la.

## Django REST Framework

Les décorateurs vus plus haut fonctionnent aussi avec DRF. Pour mettre en cache une liste publique :

```python
# viewsets.py
from rest_framework.viewsets import ReadOnlyModelViewSet
from django.utils.decorators import method_decorator
from django.views.decorators.cache import cache_page
from .models import Product
from .serializers import ProductSerializer

@method_decorator(cache_page(60 * 5), name="list")
class ProductViewSet(ReadOnlyModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
```

Seule l'action `list` est mise en cache, le détail d'un produit est recalculé à chaque appel. La négociation de contenu est prise en compte : DRF ajoute `Accept` à l'en-tête `Vary`, et la réponse JSON comme la page HTML de l'API navigable ont chacune leur entrée dans le cache (testé avec DRF 3.18).

## Vérifier que le cache sert

En développement, Django Debug Toolbar a un panneau "Cache" qui liste les appels au cache de chaque requête. Dans le code, on peut aussi tracer les succès et les échecs autour d'un calcul coûteux :

```python
import logging
from django.core.cache import cache

log = logging.getLogger(__name__)

def expensive():
    key = "exp:val"
    val = cache.get(key)
    if val is None:
        log.info("cache MISS: %s", key)
        val = compute()
        cache.set(key, val, 300)
    else:
        log.info("cache HIT: %s", key)
    return val
```

Pour mesurer le gain réel, un profileur (`cProfile`, django-silk) avant et après la mise en cache vaut mieux qu'une impression.

## Voir aussi

- [Comment ajouter un cache à une application Flask]({% post_url 2025-09-14-Comment-utiliser-un-cache-avec-Flask %})
- [Comment ajouter du cache à une application Spring Boot]({% post_url 2025-11-08-Comment-ajouter-du-cache-a-une-application-Spring-Boot %})
- [Déboguer les requêtes SQL et problèmes N+1 dans Django]({% post_url 2025-12-21-Deboguer-les-requetes-SQL-et-problemes-N-plus-1-dans-Django %})
- [Accélérer Django avec la compression HTTP]({% post_url 2025-12-13-Accelerer-Django-avec-la-compression-GZip %})
- [Documentation de Django sur le cache](https://docs.djangoproject.com/en/5.2/topics/cache/)
- [django-redis](https://github.com/jazzband/django-redis)
