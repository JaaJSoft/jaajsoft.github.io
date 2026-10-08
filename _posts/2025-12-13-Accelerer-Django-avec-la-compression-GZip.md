---
layout: article
title: "Accélérer Django avec la compression HTTP"
description: "Activer la compression HTTP GZip dans Django avec GZipMiddleware, la combiner avec WhiteNoise ou un reverse proxy, et comparer avec Brotli."
tags:
  - python
  - django
  - performance
  - http
  - compression
  - gzip
author: Pierre Chopinet
---

Les réponses texte d'une application Django (HTML, JSON, CSS, JavaScript) se compressent très bien : sur une page de test qui affiche un tableau de 100 articles, la réponse passe de 25 321 octets à environ 1 200 octets avec gzip. Dans ce tutoriel, nous allons activer la compression dans Django, vérifier qu'elle fonctionne, puis voir quand la confier à WhiteNoise ou au reverse proxy.
<!--more-->

Dans cet article :
- Activer GZipMiddleware
- Vérifier que la compression fonctionne
- Quelles réponses sont compressées
- Les fichiers statiques avec WhiteNoise
- Compresser au niveau du reverse proxy
- Brotli

Pré-requis : connaître les réglages et les middlewares de Django. Les exemples ont été testés avec Django 5.2.

## Activer GZipMiddleware

Django fournit un middleware de compression, `GZipMiddleware`. Il suffit de l'ajouter à `MIDDLEWARE`, tôt dans la liste :

```python
# settings.py
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.middleware.gzip.GZipMiddleware",  # Ajouter ici
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]
```

La documentation demande de le placer avant tout middleware qui lit ou modifie le corps de la réponse. Comme les réponses remontent la liste de bas en haut, il passe ainsi après eux et compresse la version finale. `SecurityMiddleware` peut rester devant : la documentation conseille de le garder en haut de la liste quand la redirection vers HTTPS est activée, pour ne pas faire tourner les autres middlewares avant la redirection.

Si vous utilisez aussi le cache par site, `UpdateCacheMiddleware` doit rester au-dessus de `GZipMiddleware`. La réponse est alors mise en cache déjà compressée, avec une entrée pour les clients qui acceptent gzip et une autre pour les autres, et elle n'est pas recompressée à chaque requête.

Côté sécurité, la compression expose à l'attaque BREACH, qui cherche à retrouver un secret présent dans une page compressée (un jeton CSRF par exemple) en observant la taille des réponses. Django s'en protège de deux façons : le jeton CSRF des formulaires est masqué différemment à chaque réponse, et depuis Django 4.2, `GZipMiddleware` ajoute jusqu'à 100 octets aléatoires à chaque réponse compressée. La taille d'une même page varie donc d'une requête à l'autre : entre 1 176 et 1 275 octets sur 200 appels de la page de test. Évitez malgré tout de renvoyer dans une même réponse compressée un secret et des données contrôlées par l'utilisateur.

Pour compresser une seule vue sans activer le middleware, il existe aussi le décorateur `gzip_page` :

```python
from django.views.decorators.gzip import gzip_page

@gzip_page
def ma_vue(request):
    ...
```

## Vérifier que la compression fonctionne

Le plus rapide est de regarder les en-têtes de la réponse avec curl :

```bash
curl -s -o /dev/null -D - -H "Accept-Encoding: gzip" http://127.0.0.1:8000/
```

La commande fait un vrai `GET`, jette le corps (`-o /dev/null`) et affiche les en-têtes (`-D -`). Sur la page de test servie par Gunicorn, on obtient :

```
HTTP/1.1 200 OK
Server: gunicorn
Date: Wed, 07 Oct 2026 20:05:40 GMT
Connection: close
Content-Type: text/html; charset=utf-8
X-Frame-Options: DENY
Content-Length: 1179
Vary: Accept-Encoding
Content-Encoding: gzip
X-Content-Type-Options: nosniff
Referrer-Policy: same-origin
Cross-Origin-Opener-Policy: same-origin
```

`Content-Encoding: gzip` confirme la compression et `Content-Length` donne la taille transférée. `Vary: Accept-Encoding` indique aux caches intermédiaires (CDN, proxy) de garder une version par valeur de l'en-tête `Accept-Encoding`.

Évitez `curl -I` pour ce test : cette option envoie une requête `HEAD`, et certains serveurs ne compressent pas les réponses aux requêtes `HEAD`. C'est le cas de Nginx : quand c'est lui qui compresse, `curl -I` n'affiche pas `Content-Encoding`, même si la compression est active.

Pour comparer les tailles, l'option `-w '%{size_download}'` affiche le nombre d'octets reçus :

```bash
curl -s -o /dev/null -w '%{size_download}\n' http://127.0.0.1:8000/
curl -s -o /dev/null -w '%{size_download}\n' -H 'Accept-Encoding: gzip' http://127.0.0.1:8000/
```

```
25321
1238
```

Attention à l'option `--compressed` de curl : elle décompresse la réponse avant de l'écrire. `curl --compressed http://127.0.0.1:8000/ -o page.html.gz` produit donc un fichier HTML de 25 321 octets, non compressé malgré son extension.

On peut faire la même vérification en Python avec requests :

```python
import requests

url = "http://127.0.0.1:8000/"
response = requests.get(url, headers={"Accept-Encoding": "gzip"})

print("Content-Encoding :", response.headers.get("Content-Encoding"))
print("Taille transférée :", response.headers.get("Content-Length"), "octets")
print("Taille décompressée :", len(response.content), "octets")
```

Ce qui donne :

```
Content-Encoding : gzip
Taille transférée : 1252 octets
Taille décompressée : 25321 octets
```

requests décompresse gzip automatiquement : `response.content` contient la page décompressée, ce n'est donc pas lui qu'il faut mesurer. La taille transférée se lit dans l'en-tête `Content-Length`, absent pour les réponses en streaming.

Dans le navigateur, l'onglet Réseau des outils de développement (F12) donne les mêmes informations : les en-têtes de réponse, dont `Content-Encoding`, et la taille transférée à côté de la taille réelle de la ressource.

## Quelles réponses sont compressées

`GZipMiddleware` ne regarde pas le `Content-Type` et n'a pas de liste de types à compresser. Il compresse une réponse dès que le client annonce `gzip` dans `Accept-Encoding`, qu'elle fait au moins 200 octets et qu'elle n'a pas déjà d'en-tête `Content-Encoding`. Pour une réponse classique, il garde l'original si la version compressée n'est pas plus petite. Les réponses en streaming (`StreamingHttpResponse`, `FileResponse`) sont compressées au fil de l'eau, sans ce contrôle de taille.

Voici ce que donnent quelques vues de test, avec `Accept-Encoding: gzip` :

| Réponse                                 | Sans compression | Avec GZipMiddleware |
|-----------------------------------------|------------------|---------------------|
| Texte de 150 octets                     | 150              | 150 (non compressé) |
| Page HTML, tableau de 100 articles      | 25 321           | environ 1 200       |
| JSON, liste de 100 articles             | 9 034            | environ 760         |
| `StreamingHttpResponse` de 1 000 lignes | 9 890            | environ 2 000       |
| 5 000 octets aléatoires (`image/jpeg`)  | 5 000            | 5 000 (non compressé) |

Les tailles compressées varient d'une requête à l'autre, jusqu'à une centaine d'octets, à cause des octets aléatoires. Les données déjà compressées (images, vidéos, archives) ne gagnent rien : le middleware s'en rend compte et renvoie l'original, après avoir quand même dépensé du CPU pour essayer.

Le seuil de 200 octets est écrit en dur dans la méthode `process_response`. `GZipMiddleware` n'a pas d'attribut `min_length`, et en définir un dans une sous-classe ne change rien. Pour un autre seuil, il faut surcharger la méthode :

```python
# myapp/middleware.py
from django.middleware.gzip import GZipMiddleware

class CustomGZipMiddleware(GZipMiddleware):
    min_length = 1024  # notre propre seuil (1 Ko)

    def process_response(self, request, response):
        # On applique notre seuil AVANT de déléguer au comportement standard.
        if not response.streaming and len(response.content) < self.min_length:
            return response
        return super().process_response(request, response)
```

Puis on remplace `django.middleware.gzip.GZipMiddleware` par `myapp.middleware.CustomGZipMiddleware` dans `MIDDLEWARE`. Avec ce middleware, une réponse de 600 octets n'est plus compressée, alors que `GZipMiddleware` la réduisait à une centaine d'octets.

## Les fichiers statiques avec WhiteNoise

Si WhiteNoise sert vos fichiers statiques, il sait les compresser à l'avance. Avec le stockage `CompressedManifestStaticFilesStorage`, `collectstatic` crée une version `.gz` de chaque fichier compressible, et une version `.br` si le paquet Brotli est installé (`pip install whitenoise[brotli]`). WhiteNoise envoie ensuite directement la bonne version au navigateur, sans rien compresser pendant la requête.

```python
# settings.py
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "whitenoise.middleware.WhiteNoiseMiddleware",
    "django.middleware.gzip.GZipMiddleware",  # Pour les réponses dynamiques
    # ...
]

# Compression et noms versionnés pour les statiques
STORAGES = {
    "default": {"BACKEND": "django.core.files.storage.FileSystemStorage"},
    "staticfiles": {"BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage"},
}
```

Pour la feuille de style de l'administration de Django (`admin/css/base.css`), WhiteNoise envoie ainsi 5 078 octets en gzip et 4 341 en Brotli, au lieu de 22 285.

Placez `WhiteNoiseMiddleware` au-dessus de `GZipMiddleware`, comme le demande sa documentation : les fichiers statiques sont alors servis sans jamais atteindre `GZipMiddleware`. Dans l'ordre inverse, les fichiers que WhiteNoise n'a pas compressés, comme les images, passent par `GZipMiddleware` sous forme de réponses en streaming, compressées sans contrôle de taille. Sur une image PNG de test de 12 420 octets, la réponse grossit alors à 12 519 octets et perd son en-tête `Content-Length`.

## Compresser au niveau du reverse proxy

Si un Nginx ou un Caddy est déjà devant Django, il peut se charger de la compression : les workers de Django sont déchargés de ce travail, et la configuration est commune à toutes les applications derrière le même proxy. Sans reverse proxy, ou si vous ne contrôlez pas celui de votre hébergeur, `GZipMiddleware` fait très bien l'affaire.

Activer les deux ne compresse pas deux fois, car Nginx laisse passer telle quelle une réponse qui a déjà un en-tête `Content-Encoding`. Mais c'est alors Django qui fait le travail : choisissez l'un ou l'autre.

Avec Nginx :

```nginx
# /etc/nginx/sites-available/myapp
server {
    listen 80;
    server_name votre-site.com;

    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;
    gzip_min_length 1024;
    gzip_comp_level 6;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Contrairement à Django, Nginx filtre par type de contenu avec `gzip_types`. Les réponses `text/html` sont toujours compressées, inutile de les ajouter à la liste. `gzip_min_length` se base sur l'en-tête `Content-Length` de la réponse.

Avec Caddy, la compression n'est pas active par défaut : il faut la directive `encode`, qui active zstd et gzip quand on ne précise pas de format :

```
votre-site.com {
    encode
    reverse_proxy localhost:8000
}
```

## Brotli

Brotli est un algorithme de compression publié par Google en 2015, qui donne en général des fichiers plus petits que gzip. Les navigateurs ne le demandent qu'en HTTPS : en HTTP simple, ils n'annoncent pas `br` dans `Accept-Encoding`.

Django n'a pas de middleware Brotli, il faut un paquet tiers comme `django-brotli` :

```bash
pip install django-brotli
```

```python
# settings.py
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.middleware.gzip.GZipMiddleware",       # Fallback gzip
    "django_brotli.middleware.BrotliMiddleware",   # Brotli en priorité
    # ...
]
```

L'ordre est important. Les middlewares traitent la réponse de bas en haut : placé sous `GZipMiddleware`, `BrotliMiddleware` passe en premier. Si le client accepte `br`, la réponse est compressée en Brotli et reçoit `Content-Encoding: br`, et `GZipMiddleware`, qui voit cet en-tête, la laisse telle quelle. Si le client n'accepte pas Brotli, `BrotliMiddleware` ne fait rien et `GZipMiddleware` compresse en gzip. Dans l'ordre inverse, gzip passe d'abord et Brotli n'est plus jamais utilisé pour les clients qui acceptent les deux.

Sur la page de test, la réponse fait 859 octets en Brotli contre environ 1 200 en gzip, et 478 octets contre environ 760 pour le JSON.

Attention, `django-brotli` (testé en version 0.4.0) lit entièrement les réponses en streaming et les décode en UTF-8 avant de les compresser. Une vue qui renvoie un fichier binaire avec `FileResponse` (une image, un PDF) plante alors avec une `UnicodeDecodeError` dès que le client accepte `br`.

Côté Nginx, Brotli demande le module tiers `ngx_brotli`, qu'il faut compiler ou charger en plus de Nginx :

```nginx
brotli on;
brotli_types text/plain text/css application/json application/javascript text/xml application/xml;
brotli_comp_level 6;
```

Caddy, lui, ne fait pas de Brotli à la volée : sa directive `encode` ne connaît que gzip et zstd.

## Voir aussi

- [Comment dockeriser une application Django]({% post_url 2025-10-25-Comment-dockeriser-une-application-Django %})
- [Comment ajouter du cache à une application Django]({% post_url 2025-11-01-Comment-ajouter-du-cache-a-une-application-Django %})
- [Python : Comment faire des requêtes HTTP avec requests]({% post_url 2020-05-22-Comment-faire-des-requetes-http-en-python-avec-requests %})
- [Documentation de Django sur GZipMiddleware](https://docs.djangoproject.com/en/5.2/ref/middleware/#module-django.middleware.gzip)
- [Documentation de WhiteNoise](https://whitenoise.readthedocs.io/)
