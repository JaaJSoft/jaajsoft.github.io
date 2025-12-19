---
layout: article
title: "Python : Comment utiliser les sessions avec requests pour optimiser vos appels HTTP"
author: Pierre Chopinet
tags:
  - python
  - http
  - api
  - rest
  - requests
  - performance
  - authentification
---

Les sessions (`requests.Session`) apportent un vrai gain de performance et de simplicité quand vous faites plusieurs requêtes vers une même API : elles réutilisent les connexions (keep‑alive), partagent automatiquement les cookies, en‑têtes et authentifications, et permettent de configurer des stratégies de retries.
<!--more-->

Dans cet article, on voit :

- Pourquoi et quand utiliser une session
- Comment partager des headers, cookies et authentifications
- Activer des retries et le pooling via `HTTPAdapter`
- Gérer les timeouts, proxys, SSL
- Les bonnes pratiques (context manager, thread‑safety, pièges courants)


> Astuce : si vous débutez avec requests, commencez par l’article de base « [Comment faire des requêtes HTTP avec requests]({% post_url 2020-05-22-Comment-faire-des-requetes-http-en-python-avec-requests %}) », puis revenez ici pour optimiser vos appels.

## 1) Pourquoi utiliser requests.Session ?

Sans session, chaque appel `requests.get/post/...` ouvre une nouvelle connexion TCP/TLS, ce qui coûte du temps (handshake) et des ressources.
Une `Session` :
- Réutilise les connexions grâce au keep‑alive (connection pooling)
- Conserve automatiquement cookies et certains en‑têtes entre requêtes
- Permet de définir une authentification une fois pour toutes
- Centralise la configuration (timeouts par défaut, proxies, SSL, retries, User‑Agent, etc.)

Résultat : moins de latence, code plus concis, et meilleure robustesse.

### Session de base

```python
import requests

with requests.Session() as s:
    r1 = s.get("https://httpbin.org/cookies/set?session=jaaj")
    r1.raise_for_status()
    # Le cookie est conservé et renvoyé automatiquement à la prochaine requête
    r2 = s.get("https://httpbin.org/cookies")
    print(r2.json())  # {"cookies": {"session": "jaaj"}}
```

## 2) Partager des en‑têtes (headers) et une authentification

Vous pouvez définir des headers et une auth sur la session ; ils seront appliqués à toutes les requêtes (et pourront être surchargés au cas par cas).

```python
import requests
from requests.auth import HTTPBasicAuth

with requests.Session() as s:
    s.headers.update({
        "User-Agent": "jaaj.dev-tutoriel/1.0",
        "Accept": "application/json",
    })
    s.auth = HTTPBasicAuth("user", "pass")  # ou s.auth = ("user", "pass")

    r = s.get("https://api.example.com/me", timeout=10)
    r.raise_for_status()
    print(r.json())
```

- `s.headers.update(...)` permet de définir un jeu d’en‑têtes par défaut.
- `s.auth` évite de répéter l’authentification à chaque appel.

## 3) Cookies persistants et RequestsCookieJar

La session gère un `cookiejar` persistant tant que la session vit.

```python
import requests

with requests.Session() as s:
    s.cookies.set("locale", "fr-FR", domain="example.com")
    s.get("https://example.com/")  # le cookie sera envoyé si le domaine correspond

    # Accéder aux cookies
    for c in s.cookies:
        print(c.name, c.value)
```

Pratique pour des parcours d’authentification basés sur cookies (login form) ou des données plus persistantes (ex. token, paniers, configuration).

## 4) Retries et pooling via HTTPAdapter

Par défaut, requests n’applique pas de retries automatiques. En session, on peut brancher un `HTTPAdapter` pour :

- Définir un pool de connexions maximum par hôte (pooling)
- Configurer des retries sur erreurs réseau ou codes 5xx/429

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

retry_strategy = Retry(
    total=3,                # 3 tentatives au total
    backoff_factor=0.5,     # délai exponentiel: 0.5, 1, 2, ...
    status_forcelist=[429, 500, 502, 503, 504],
    allowed_methods=["HEAD", "GET", "OPTIONS", "POST"],  # POST si idempotent côté serveur
    raise_on_status=False,
)

adapter = HTTPAdapter(max_retries=retry_strategy, pool_connections=20, pool_maxsize=20)

with requests.Session() as s:
    s.mount("https://", adapter)
    s.mount("http://", adapter)

    r = s.get("https://api.example.com/resource", timeout=10)
    r.raise_for_status()
    print(r.json())
```

Notes :
- `allowed_methods` s’applique aux méthodes idempotentes. N’activez `POST` que si votre API le supporte sans effet secondaire.
- `pool_connections` et `pool_maxsize` contrôlent la taille du pool.

## 5) Timeouts & proxies

Toujours mettre un timeout pour éviter de bloquer indéfiniment.

```python
with requests.Session() as s:
    # Proxy (par ex. réseau d’entreprise)
    s.proxies.update({
        "http": "http://proxy.local:3128",
        "https": "http://proxy.local:3128",
    })

    # Timeout par appel (connect, read)
    r = s.get("https://example.com/slow", timeout=(3.05, 10))
    r.raise_for_status()
```

## 6) Bonnes pratiques

- Utilisez un context manager : `with requests.Session() as s:` pour garantir la fermeture propre des connexions (appel implicite à `close()`).
- Définissez des timeouts à chaque requête (ou enveloppez vos méthodes pour un timeout par défaut).
- Centralisez headers/auth/proxies au niveau session.
- `Session` et thread‑safety : une même instance peut être utilisée en lecture simultanée avec prudence, mais l’API requests ne garantit pas une thread‑safety totale. Le plus sûr est d’avoir une session par thread ou d’utiliser un pool de sessions.
- N’exposez pas de session globale modifiable dans une librairie ; préférez l’injection (passer la session) ou une fabrique qui crée une session configurée.
- Pensez à journaliser (`logging`) les URLs cibles, statuts et latences.

## 7) Modèle de client réutilisable

Exemple d’un petit client API qui encapsule une session configurée :

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

class ApiClient:
    def __init__(self, base_url: str, token: str | None = None, timeout: float = 10.0):
        self.base_url = base_url.rstrip("/")
        self.timeout = timeout
        self.session = requests.Session()

        # Headers par défaut
        self.session.headers.update({
            "User-Agent": "jaaj.dev-api-client/1.0",
            "Accept": "application/json",
        })
        if token:
            self.session.headers["Authorization"] = f"Bearer {token}"

        # Retries + pooling
        retry = Retry(total=3, backoff_factor=0.5, status_forcelist=[429, 500, 502, 503, 504])
        adapter = HTTPAdapter(max_retries=retry, pool_connections=20, pool_maxsize=20)
        self.session.mount("https://", adapter)
        self.session.mount("http://", adapter)

    def _url(self, path: str) -> str:
        return f"{self.base_url}/{path.lstrip('/')}"

    def get(self, path: str, **kwargs):
        timeout = kwargs.pop("timeout", self.timeout)
        r = self.session.get(self._url(path), timeout=timeout, **kwargs)
        r.raise_for_status()
        return r

    def post(self, path: str, **kwargs):
        timeout = kwargs.pop("timeout", self.timeout)
        r = self.session.post(self._url(path), timeout=timeout, **kwargs)
        r.raise_for_status()
        return r

    def close(self):
        self.session.close()

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc, tb):
        self.close()

# Utilisation
with ApiClient("https://api.example.com", token="...") as api:
    me = api.get("me").json()
    orders = api.get("orders").json()
    print(me, len(orders))
```

---

En résumé, `requests.Session` est un incontournable pour toute intégration HTTP sérieuse : plus rapide, plus fiable et plus facile à maintenir. Combinez‑la avec des `HTTPAdapter` pour les retries, définissez des timeouts, et utilisez le context manager pour une gestion propre des ressources.

Pour aller plus loin :
- [Comment faire des requêtes HTTP avec requests]({% post_url 2020-05-22-Comment-faire-des-requetes-http-en-python-avec-requests %})
- [Python : Comment utiliser les différents modes d'authentification avec requests]({% post_url 2025-09-05-Comment-utiliser-l-authentification-avec-requests %})
- Documentation officielle requests : [https://requests.readthedocs.io](https://requests.readthedocs.io)
