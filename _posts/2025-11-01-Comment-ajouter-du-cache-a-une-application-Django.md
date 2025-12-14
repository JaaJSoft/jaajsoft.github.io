---
layout: article
title: "Comment ajouter du cache à une application Django"
author: Pierre Chopinet
tags:
  - python
  - django
  - cache
  - performance
  - redis
  - memcached
  - tutoriel
---

Accélérer une app Django ne passe pas uniquement par le code: un bon cache peut diviser la charge serveur par 10 et réduire drastiquement la latence. Dans ce guide, on passe en revue les niveaux de cache offerts par Django, avec des exemples concrets pour chaque usage.
<!--more-->

Objectifs:
- Choisir le backend de cache adapté (mémoire locale, fichier, base de données, Memcached, Redis)
- Configurer `CACHES` (par défaut + Redis/Memcached)
- Utiliser les différents niveaux de cache: per‑view, per‑site (middleware), fragments de template, API bas niveau
- Invalider proprement: TTL, suppression ciblée, versioning, signaux
- Gérer les cas réels (DRF, pages par utilisateur, i18n), pièges et bonnes pratiques

Prérequis :
- Django 4.2+ (ou 5.x)
- Notions de vues/templates

Avant de configurer quoi que ce soit, commençons par choisir le backend de cache le plus adapté à votre contexte.

---

## 1) Choisir un backend de cache

Django propose plusieurs backends. Voici quand utiliser quoi, avec configuration minimale.

### a) Mémoire locale
- Avantages : zéro dépendance, très simple, rapide.
- Limites : non partagé entre processus/containers, pas persistant.

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

### b) Fichier
- Avantages : persistant, simple.
- Limites : I/O disque, moins adapté à forte concurrence.

```python
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.filebased.FileBasedCache",
        "LOCATION": BASE_DIR / "django_cache",  # dossier existant et accessible
        "TIMEOUT": 600,
    }
}
```

### c) Base de données
- Avantages : pas de service externe.
- Limites : met plus de charge sur la DB; latences supérieures à Redis/Memcached.

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

### d) Memcached
- Avantages : mémoire partagée très rapide; mature.
- Limites : valeurs limitées en taille (~1 Mo par défaut), pas de types riches.

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

### e) Redis (recommandé pour la plupart des apps)
- Avantages : partagé, rapide, TTL, incr/decr, locks; très courant en prod.
- Limites : dépendance externe, gestion/ops.

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
            # Ne pas crasher si Redis est indisponible (log et fallback)
            "IGNORE_EXCEPTIONS": True,
        },
        "TIMEOUT": 300,
        "KEY_PREFIX": "myapp",
    }
}
```

En partant du choix du backend, voyons l’approche la plus simple à mettre en place côté vues.

---

## 2) Cache per‑view (le plus simple à adopter)

Idéal pour des pages « identiques pour tous » (ex: page d’accueil publique).

```python
# views.py
from django.views.generic import TemplateView
from django.utils.decorators import method_decorator
from django.views.decorators.cache import cache_page, cache_control, vary_on_headers

@method_decorator(cache_page(60 * 15), name="dispatch")  # 15 min
@method_decorator(cache_control(public=True), name="dispatch")
class HomeView(TemplateView):
    template_name = "home.html"
```

Routes :
```python
# urls.py
from django.urls import path
from .views import HomeView
urlpatterns = [
    path("", HomeView.as_view(), name="home"),
]
```

- `cache_page(timeout, key_prefix=None)` crée/retourne la réponse depuis le cache.
- `cache_control(public=True)` ajoute des en‑têtes HTTP utiles côté CDN/navigateur.
- Pour varier par langue ou agent, utilisez `vary_on_headers`:

```python
@method_decorator(vary_on_headers("Accept-Language"), name="dispatch")
class HomeView(...):
    ...
```

> Attention : pour des pages dépendantes de l’utilisateur connecté, le per‑view naïf n’est pas adapté (risque de servir la page d’un autre). Voir section « Par utilisateur ».

---

## 3) Cache per‑site (middlewares)

Met en cache toutes les réponses (GET/HEAD) si possible. Pratique pour un site essentiellement statique.

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

Exclure certaines vues :

```python
# views.py
from django.views.decorators.cache import never_cache

@never_cache
def admin_dashboard(request):
    ...
```

> Le per‑site convient aux contenus publics. Évitez pour des pages personnalisées par utilisateur (ou utilisez un CDN avec règles précises).

Quand seule une portion d’une page est coûteuse, on peut cibler ce périmètre : passons au cache de fragments dans les templates.

---

## 4) Cache de fragments dans les templates

Parfait pour une « sidebar », un bloc de navigation coûteux, ou un composant externe.

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

- La clé de cache inclut ici `user.pk` pour différencier par utilisateur.
- Gardez les fragments assez gros pour amortir le coût, mais pas trop (évitez
  d'avoir un trop grand nombre de clés de cache différentes qui pourraient
  surcharger la mémoire).

Pour des besoins plus fins, passons maintenant à l’API bas niveau qui permet de lire/écrire directement des clés.

---

## 5) API bas niveau : cache.get/set/get_or_set/...

Quand vous gérez vos propres clés/objets.

```python
# services.py
from django.core.cache import cache

KEY = "stats:homepage"

def get_home_stats():
    def _compute():
        # Simule un calcul/IO lourd
        return {"articles": 42, "users": 1337}

    # get_or_set calcule et stocke si absent
    return cache.get_or_set(KEY, _compute, timeout=300)
```

Opérations utiles :

```python
cache.set("foo", {"x": 1}, timeout=60)
val = cache.get("foo")                       # -> {"x": 1}
cache.add("foo", 2, timeout=60)              # n’écrase pas si existe déjà
cache.incr("counter", delta=1)               # Redis/Memcached
cache.decr("counter", delta=2)
cache.delete("foo")
cache.delete_many(["k1", "k2"])
cache.get("k", default=None, version=2)      # versioning
# Mettre à jour le TTL (si supporté)
try:
    cache.touch("stats:homepage", timeout=120)
except NotImplementedError:
    pass
```

> Sérialisation : Django sérialise les données en utilisant Pickle par défaut.
> Avec Redis, vous pouvez configurer un compresseur pour réduire la taille (voir
`OPTIONS["COMPRESSOR"]`).

Maintenant que vous savez manipuler le cache au plus près du code, voyons comment l’invalider proprement (TTL, suppression ciblée, versioning, signaux).

---

## 6) Invalidation : TTL, suppression ciblée, versioning, signaux

- TTL : fixez des `TIMEOUT` réalistes (ex : 5–15 min pour du contenu éditorial).
- Suppression ciblée : quand une ressource change.

```python
from django.core.cache import cache

def invalidate_article(article_id: int):
    cache.delete(f"article:{article_id}")
```

- Par motif (Redis + django‑redis):

```python
from django_redis import get_redis_connection

r = get_redis_connection("default")
for key in r.scan_iter("user:*"):
    r.delete(key)
# ou plus simple si disponible:
# cache.delete_pattern("user:*")
```

- Versioning de clés : pratique lors d’un déploiement changeant le format :

```python
CACHE_VERSION = 2  # settings.py
# utilisation
cache.get("home", version=CACHE_VERSION)
```

- Signaux : invalider au `post_save`/`post_delete`.

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

---

## 7) Patterns courant de cache pour la gestion des utilisateurs, i18n et pagination

### a) Par utilisateur

Évitez de mettre en cache la page complète. Cachez les fragments ou des data :

```python
from django.core.cache import cache

def get_user_dashboard(user):
    key = f"dash:{user.pk}"
    return cache.get_or_set(key, lambda: compute_dashboard(user), 300)
```

Pour des vues per‑view conditionnelles, variez sur cookies/headers avec prudence :

```python
from django.views.decorators.vary import vary_on_cookie

@vary_on_cookie
@cache_page(300)
def public_but_personalized(request):
    ...
```

> Attention : `vary_on_cookie` multiplie les variantes dans le cache (risque d’explosion du nombre de clés). Préférez les fragments ciblés.

### b) Internationalisation

Variez sur `Accept-Language` ou utilisez des clés incluant le code langue :

```python
from django.utils.translation import get_language
key = f"home:{get_language()}"
```

### c) Pagination/tri

Incluez les paramètres de requête dans la clé (ou laissez le middleware/vary s’en charger) :

```python
page = request.GET.get("page", "1")
sort = request.GET.get("sort", "-date")
key = f"list:{page}:{sort}"
```

---

## 8) Django REST Framework (DRF)

Les options pour cacher présentés précédemment fonctionnent aussi avec DRF.
Par exemple mettre en cache les listes publiques est souvent rentable :

```python
# viewsets.py
from rest_framework.viewsets import ReadOnlyModelViewSet
from django.utils.decorators import method_decorator
from django.views.decorators.cache import cache_page
from django.views.decorators.vary import vary_on_headers
from .models import Product
from .serializers import ProductSerializer

@method_decorator(cache_page(60 * 5), name="list")
@method_decorator(vary_on_headers("Accept-Language"), name="list")
class ProductViewSet(ReadOnlyModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
```

---

## 9) Observabilité, tests et mise au point

- `django-debug-toolbar` possède un panneau « Cache » utile en dev.
- Loggez les hits/miss autour d’un bloc coûteux :

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

- Mesurez : un profilage simple (ex : `django-silk`, `cProfile`) permet de vérifier les gains.

---

## 10) Pièges et bonnes pratiques

- Ne mettez pas en cache des données sensibles (tokens, infos personnelles) dans un cache partagé.
- Invalidez au plus près des changements de données (signaux) et gardez des TTL raisonnables.
- Évitez les clés déduites de l’URL brute si elle contient des IDs sensibles ; préférez des clés explicites.
- Prenez garde aux variantes (`Vary`) non maîtrisées : `vary_on_cookie` peut exploser le nombre d’entrées.
- Pour les déploiements multi-process/containers, évitez d'utiliser
  `LocMemCache` car chaque process aura son propre cache isolé.
- Avec Memcached: évitez de dépasser ~1 Mo par valeur; sérialisez des données légères.

---

## Cheatsheet

- Config Redis:
  ```python
  CACHES = {"default": {"BACKEND": "django_redis.cache.RedisCache", "LOCATION": "redis://..."}}
  ```
- Per‑view:
  ```python
  @method_decorator(cache_page(900), name="dispatch")
  class MyView(TemplateView): ...
  ```
- Per‑site (middlewares): `UpdateCacheMiddleware` en premier, `FetchFromCacheMiddleware` en dernier.
- Fragment:
  {% raw %}
  ```django
  {% load cache %}{% cache 600 name arg %}...{% endcache %}
  ```
  {% endraw %}
- Bas niveau:
  ```python
  cache.get/set/get_or_set/delete/incr/decr/touch
  ```
- Invalidation ciblée: `cache.delete("k")`, motif Redis `scan_iter("prefix:*")`.

---

## Conclusion

Le cache Django offre plusieurs leviers complémentaires : commencez par le per‑view sur vos pages publiques, ajoutez du fragment cache pour les blocs coûteux, utilisez Redis comme backend partagé, et mettez en place une stratégie d’invalidation claire. Vous gagnerez en latence, en coûts infra, et en sérénité.

---

## Pour aller plus loin

- [Comment dockeriser une application Django]({% post_url 2025-10-25-Comment-dockeriser-une-application-Django %})
- [Accélérer Django avec la compression GZip]({% post_url 2025-12-13-Accelerer-Django-avec-la-compression-GZip %})
- [Documentation officielle Django (cache)](https://docs.djangoproject.com/en/stable/topics/cache/)
