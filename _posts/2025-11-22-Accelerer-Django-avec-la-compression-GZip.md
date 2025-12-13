---
layout: article
title: "Accélérer Django avec la compression HTTP GZip"
date: 2025-11-22
author: Pierre Chopinet
tags:
  - python
  - django
  - performance
  - http
  - compression
  - gzip
  - tutoriel
---

La compression HTTP permet de réduire drastiquement la taille des réponses envoyées par votre serveur (HTML, JSON, CSS, JavaScript). Une page de 500 Ko peut facilement passer à 50 Ko après compression GZip, ce qui accélère le chargement, réduit la bande passante et améliore l'expérience utilisateur, surtout sur mobile.
<!--more-->

Dans ce guide, vous allez apprendre à :

- Activer la compression GZip dans Django avec le middleware intégré
- Configurer les seuils et types de contenu à compresser
- Vérifier que la compression fonctionne correctement
- Comprendre quand utiliser GZip côté Django vs côté reverse proxy (Nginx, Caddy)
- Éviter les pièges courants (double compression, types de contenu incompatibles)

Prérequis :
- Django 4.2+ (fonctionne aussi avec Django 3.x et 5.x)
- Connaissances de base de Django (settings, middlewares)

---

## 1) Pourquoi activer la compression HTTP ?

### Avantages

- **Réduction de la bande passante** : économie de 60 à 90 % sur les réponses textuelles (HTML, JSON, CSS, JS).
- **Temps de chargement réduit** : pages plus rapides, surtout sur connexions lentes (3G/4G).
- **Meilleur référencement** : Google favorise les sites rapides.
- **Coût infra réduit** : moins de données transférées = moins de facture cloud/CDN.

### Limites

- **CPU légèrement sollicité** : la compression consomme un peu de CPU côté serveur (négligeable dans 99 % des cas).
- **Fichiers déjà compressés** : inutile pour les images (JPEG, PNG, WebP), vidéos (MP4) ou archives (ZIP) déjà compressées.
- **Seuil de taille** : compresser une réponse de 10 octets n'a pas de sens (overhead supérieur au gain).

---

## 2) Activer la compression GZip dans Django

Django intègre un middleware de compression : `GZipMiddleware`. Il suffit de l'ajouter dans `MIDDLEWARE` (le plus tôt possible dans la chaîne, juste après `SecurityMiddleware`).

### Configuration minimale

```python
# settings.py
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.middleware.gzip.GZipMiddleware",  # Ajouter ici
    # ... vos autres middlewares ...
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]
```

> **Ordre important** : placez `GZipMiddleware` le plus tôt possible, mais après `SecurityMiddleware` (pour que les headers de sécurité soient traités en premier). Si vous utilisez WhiteNoise, placez `GZipMiddleware` après `WhiteNoiseMiddleware`.

### Exemple avec WhiteNoise (fichiers statiques)

Si vous utilisez WhiteNoise pour servir vos fichiers statiques (recommandé en production sans Nginx), placez-le avant `GZipMiddleware` :

```python
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "whitenoise.middleware.WhiteNoiseMiddleware",  # Statiques en premier
    "django.middleware.gzip.GZipMiddleware",       # Compression ensuite
    # ... autres middlewares ...
]
```

> **Note** : WhiteNoise peut aussi compresser les statiques à l'avance (pré-compression). Voir la section « Combiner avec WhiteNoise ».

---

## 3) Vérifier que la compression fonctionne

### Avec curl

```bash
curl -I -H "Accept-Encoding: gzip" https://votre-site.com/
```

Cherchez l'en-tête `Content-Encoding: gzip` dans la réponse :

```
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Encoding: gzip
Content-Length: 4567
```

### Avec les DevTools du navigateur

1. Ouvrez l'inspecteur (F12) > onglet **Network**.
2. Rechargez la page.
3. Cliquez sur la requête principale (document HTML).
4. Vérifiez les en-têtes de réponse : `Content-Encoding: gzip`.
5. Comparez **Size** (taille transférée) et **Content** (taille décompressée).

Exemple :
- **Content** : 523 Ko (taille originale)
- **Size** : 87 Ko (taille transférée après compression)
- **Gain** : ~83 % de réduction

### Script Python de test

```python
import requests

url = "https://votre-site.com/"
headers = {"Accept-Encoding": "gzip"}
response = requests.get(url, headers=headers)

print(f"Status: {response.status_code}")
print(f"Content-Encoding: {response.headers.get('Content-Encoding', 'none')}")
print(f"Taille compressée: {len(response.content)} octets")
print(f"Taille décompressée: {len(response.text)} octets")
```

> Voir aussi : [Comment faire des requêtes HTTP en Python avec requests]({% post_url 2020-05-22-Comment-faire-des-requetes-http-en-python-avec-requests %})

---

## 4) Configuration avancée : seuils et types de contenu

### Seuil de taille minimum

Par défaut, Django compresse toutes les réponses de plus de **200 octets**. Ce seuil est défini dans le code du middleware et n'est pas configurable via `settings.py`.

Si vous voulez modifier ce seuil, vous devez créer un middleware personnalisé :

```python
# myapp/middleware.py
from django.middleware.gzip import GZipMiddleware

class CustomGZipMiddleware(GZipMiddleware):
    min_length = 1024  # Compresser uniquement si > 1 Ko
```

Puis remplacez `django.middleware.gzip.GZipMiddleware` par `myapp.middleware.CustomGZipMiddleware` dans `MIDDLEWARE`.

### Types de contenu compressés

Django compresse automatiquement les types de contenu textuels courants :
- `text/html`
- `text/plain`
- `text/css`
- `application/json`
- `application/javascript`
- `text/xml`
- `application/xml`

**Types non compressés** (déjà binaires ou compressés) :
- Images : `image/jpeg`, `image/png`, `image/webp`, `image/gif`
- Vidéos : `video/mp4`, `video/webm`
- Archives : `application/zip`, `application/gzip`
- Polices : `font/woff2` (déjà compressé)

Le middleware détecte automatiquement le type de contenu et évite de compresser les fichiers déjà compressés.

---

## 5) Combiner avec WhiteNoise (pré-compression des statiques)

WhiteNoise offre une fonctionnalité de **pré-compression** : vos fichiers statiques (CSS, JS) sont compressés une seule fois au `collectstatic`, puis servis directement compressés (zéro CPU en prod).

### Configuration recommandée

```python
# settings.py
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "whitenoise.middleware.WhiteNoiseMiddleware",
    "django.middleware.gzip.GZipMiddleware",  # Pour les réponses dynamiques
    # ...
]

# Compression + cache des statiques (recommandé)
STATICFILES_STORAGE = "whitenoise.storage.CompressedManifestStaticFilesStorage"
```

Avec cette config :
- **WhiteNoise** : sert les fichiers statiques (CSS, JS) pré-compressés (`.gz`).
- **GZipMiddleware** : compresse les réponses dynamiques (HTML, JSON des vues Django).

> **Attention** : si WhiteNoise sert déjà vos statiques compressés, `GZipMiddleware` ne les re-compressera pas (pas de double compression).

---

## 6) GZip côté Django vs côté reverse proxy (Nginx, Caddy)

### Quand compresser côté Django ?

- Vous n'avez pas de reverse proxy (Nginx, Caddy, Traefik).
- Vous déployez sur un PaaS (Heroku, Render, Fly.io) qui ne gère pas la compression par défaut.
- Vous voulez une solution simple, sans config externe.

### Quand compresser côté Nginx/Caddy ?

- Vous avez déjà un reverse proxy en place.
- Vous voulez décharger Django du travail de compression (légère économie de CPU).
- Vous gérez plusieurs apps derrière le même proxy (factorisation de la config).

### Exemple Nginx

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

### Exemple Caddy

Caddy active la compression par défaut (gzip, zstd, brotli). Aucune config supplémentaire nécessaire :

```
votre-site.com {
    reverse_proxy localhost:8000
}
```

> **Recommandation** : si vous avez un reverse proxy, préférez compresser à ce niveau (plus performant). Sinon, `GZipMiddleware` est parfait.

---

## 7) Compression Brotli (alternative moderne à GZip)

**Brotli** est un algorithme de compression plus récent (Google, 2015) qui offre 15 à 25 % de gain supplémentaire par rapport à GZip, avec un support navigateur excellent (>95 %).

### Côté Django

Django n'intègre pas de middleware Brotli par défaut. Vous pouvez utiliser `django-brotli` :

```bash
pip install django-brotli
```

```python
# settings.py
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django_brotli.middleware.BrotliMiddleware",  # Avant GZipMiddleware
    "django.middleware.gzip.GZipMiddleware",      # Fallback si Brotli non supporté
    # ...
]
```

Le middleware Brotli détecte si le client supporte `br` (via `Accept-Encoding`) et compresse avec Brotli, sinon repasse à GZip.

### Côté Nginx

```nginx
# Nécessite le module ngx_brotli
brotli on;
brotli_types text/plain text/css application/json application/javascript text/xml application/xml;
brotli_comp_level 6;
```

### Côté Caddy

Activé par défaut (Caddy choisit automatiquement entre brotli, gzip, zstd selon le client).

---

## 8) Pièges courants et bonnes pratiques

### Pièges

- **Double compression** : si Nginx/Caddy compresse déjà, ne compressez pas côté Django (gaspillage CPU). Désactivez l'un des deux.
- **Ordre des middlewares** : `GZipMiddleware` doit être tôt dans la chaîne, mais après `SecurityMiddleware` et `WhiteNoiseMiddleware`.
- **Fichiers déjà compressés** : ne compressez pas les images, vidéos, archives (aucun gain, voire augmentation de taille).
- **Seuil trop bas** : compresser des réponses de 50 octets ajoute plus d'overhead que de gain.
- **Cache et compression** : assurez-vous que votre cache stocke bien la version compressée (ou que la compression est appliquée après le cache).

### Bonnes pratiques

- Activez la compression dès le début du projet (pas d'impact négatif).
- Combinez avec du cache pour éviter de recompresser les mêmes réponses (voir [Comment ajouter du cache à une application Django]({% post_url 2025-11-01-Comment-ajouter-du-cache-a-une-application-Django %})).
- Mesurez l'impact réel (DevTools, Lighthouse, GTmetrix).
- Préférez Brotli si vous avez un reverse proxy moderne (Nginx avec `ngx_brotli`, Caddy).
- Utilisez WhiteNoise avec `CompressedManifestStaticFilesStorage` pour les statiques.

---

## 9) Mesurer le gain de performance

### Lighthouse (Chrome DevTools)

1. Ouvrez Chrome DevTools > onglet **Lighthouse**.
2. Lancez un audit (Performance + Best Practices).
3. Cherchez « Enable text compression » dans les recommandations.
4. Comparez le score avant/après activation.

### GTmetrix / WebPageTest

- GTmetrix : https://gtmetrix.com/
- WebPageTest : https://www.webpagetest.org/

Entrez l'URL de votre site et vérifiez :
- **Transfer Size** (taille transférée)
- **Content Size** (taille décompressée)
- **Compression Ratio**

### Commande locale (avant/après)

```bash
# Sans compression (désactiver GZipMiddleware)
curl -H "Accept-Encoding: identity" https://votre-site.com/ -o page.html
ls -lh page.html

# Avec compression
curl -H "Accept-Encoding: gzip" https://votre-site.com/ --compressed -o page.html.gz
ls -lh page.html.gz
```

---

## Cheatsheet

### Activer GZip

```python
# settings.py
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.middleware.gzip.GZipMiddleware",  # Ajouter ici
    # ...
]
```

### Vérifier avec curl

```bash
curl -I -H "Accept-Encoding: gzip" https://votre-site.com/
# Chercher : Content-Encoding: gzip
```

### WhiteNoise + pré-compression

```python
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "whitenoise.middleware.WhiteNoiseMiddleware",
    "django.middleware.gzip.GZipMiddleware",
    # ...
]

STATICFILES_STORAGE = "whitenoise.storage.CompressedManifestStaticFilesStorage"
```

### Nginx (alternative)

```nginx
gzip on;
gzip_types text/plain text/css application/json application/javascript;
gzip_min_length 1024;
```

---

## Conclusion

Activer la compression GZip dans Django est trivial (une ligne dans `MIDDLEWARE`) et offre un gain substantiel de performance (50 à 90 % de réduction de bande passante). Combinez avec du cache, WhiteNoise et un reverse proxy pour maximiser la vitesse de votre application. Si vous avez déjà un Nginx/Caddy, préférez compresser à ce niveau pour économiser du CPU Django.

---

## Voir aussi

- [Comment ajouter du cache à une application Django]({% post_url 2025-11-01-Comment-ajouter-du-cache-a-une-application-Django %})
- [Comment dockeriser une application Django]({% post_url 2025-10-25-Comment-dockeriser-une-application-Django %})
- [Comment faire des requêtes HTTP en Python avec requests]({% post_url 2020-05-22-Comment-faire-des-requetes-http-en-python-avec-requests %})
- [Documentation Django (GZipMiddleware)](https://docs.djangoproject.com/en/stable/ref/middleware/#module-django.middleware.gzip)
- [WhiteNoise documentation](http://whitenoise.evans.io/)
