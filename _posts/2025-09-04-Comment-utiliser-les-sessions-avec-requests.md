---
layout: article
title: "Python : Comment utiliser les sessions avec requests"
description: "Réutiliser les connexions avec requests.Session en Python : cookies, en-têtes et authentification partagés, retries avec HTTPAdapter, timeouts et proxies."
tags:
  - python
  - http
  - api
  - rest
  - requests
  - performance
  - authentification
author: Pierre Chopinet
---

Quand un script enchaîne les appels vers une même API, chaque `requests.get()` ouvre une nouvelle connexion. Dans ce tutoriel, nous allons voir comment utiliser `requests.Session` pour réutiliser les connexions, garder les cookies d'une requête à l'autre et ne configurer qu'une fois les en-têtes, l'authentification et les retries.
<!--more-->

Dans cet article :
- Pourquoi utiliser une session
- En-têtes et authentification partagés
- Les cookies de la session
- Retries et pool de connexions avec HTTPAdapter
- Timeouts et proxies
- Vérification des certificats SSL
- Un client d'API réutilisable

Pré-requis : Python 3 et la bibliothèque requests (`pip install requests`). Si vous débutez avec requests, commencez par [Python : Comment faire des requêtes HTTP avec requests]({% post_url 2020-05-22-Comment-faire-des-requetes-http-en-python-avec-requests %}). Les exemples ont été testés avec requests 2.34.2 et urllib3 2.8.0.

## Pourquoi utiliser une session

Les fonctions `requests.get()`, `requests.post()` et leurs cousines créent une session temporaire à chaque appel, puis la ferment : chaque requête ouvre sa propre connexion TCP, avec une nouvelle négociation TLS en HTTPS.

Une session garde au contraire ses connexions ouvertes (*keep-alive*) et les réutilise tant qu'on interroge le même hôte. Elle conserve aussi les cookies renvoyés par le serveur :

```python
import requests

with requests.Session() as s:
    r1 = s.get("https://httpbin.org/cookies/set?session=jaaj")
    r1.raise_for_status()
    # Le cookie est conservé et renvoyé automatiquement à la prochaine requête
    r2 = s.get("https://httpbin.org/cookies")
    print(r2.json())
```

Ce qui donne :

```
{'cookies': {'session': 'jaaj'}}
```

Le bloc `with` ferme la session et ses connexions à la sortie, même si une exception est levée. Sans `with`, pensez à appeler `s.close()` vous-même.

## En-têtes et authentification partagés

Les en-têtes et l'authentification définis sur la session sont envoyés avec toutes ses requêtes :

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

Les paramètres passés à une requête sont fusionnés avec ceux de la session et passent devant en cas de conflit. Pour retirer ponctuellement un en-tête défini sur la session, on lui donne la valeur `None` :

```python
r = s.get("https://api.example.com/export", headers={"Accept": None}, timeout=10)
```

Les autres modes d'authentification (Bearer, OAuth, classe personnalisée...) sont détaillés dans [l'article sur l'authentification avec requests]({% post_url 2025-09-05-Comment-utiliser-l-authentification-avec-requests %}).

## Les cookies de la session

La session range ses cookies dans `s.cookies`, un `RequestsCookieJar` qui vit aussi longtemps qu'elle. On peut y ajouter des cookies à la main :

```python
import requests

with requests.Session() as s:
    s.cookies.set("locale", "fr-FR", domain="example.com")
    s.get("https://example.com/")  # le cookie sera envoyé si le domaine correspond

    # Accéder aux cookies
    for c in s.cookies:
        print(c.name, c.value)
```

Le cookie `locale` est envoyé à `example.com` et à ses sous-domaines, mais pas aux autres sites. C'est ce mécanisme qui permet de gérer une connexion par formulaire : après le `POST` sur la page de login, le cookie reçu du site est renvoyé automatiquement avec les requêtes suivantes.

Attention, un cookie passé directement à une requête avec `s.get(url, cookies={...})` n'est envoyé qu'avec cette requête : il n'est pas ajouté à la session.

## Retries et pool de connexions avec HTTPAdapter

Par défaut, requests ne relance jamais une requête qui a échoué. Pour ajouter des retries, on monte sur la session un `HTTPAdapter` configuré avec un objet `Retry` d'urllib3 :

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

retry_strategy = Retry(
    total=3,                # jusqu'à 3 nouvelles tentatives
    backoff_factor=0.5,     # attentes de 0 s, 1 s puis 2 s
    status_forcelist=[429, 500, 502, 503, 504],
    allowed_methods=["HEAD", "GET", "OPTIONS", "POST"],  # POST seulement si l'API le permet
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

`total=3` autorise trois nouvelles tentatives, soit quatre requêtes au maximum. La première nouvelle tentative part tout de suite, puis l'attente double à chaque fois à partir de `2 * backoff_factor` : avec `0.5`, on attend donc 0 s, 1 s puis 2 s. Si la réponse contient un en-tête `Retry-After` (typiquement avec un code 429 ou 503), urllib3 attend le délai demandé par le serveur au lieu d'appliquer ce calcul.

`status_forcelist` liste les codes HTTP qui déclenchent une nouvelle tentative. Les erreurs de connexion (serveur injoignable, connexion refusée) sont retentées quelle que soit la méthode, puisque la requête n'est jamais partie. Dans les autres cas, urllib3 ne retente par défaut que les méthodes idempotentes (`GET`, `HEAD`, `PUT`, `DELETE`, `OPTIONS`, `TRACE`). En passant `allowed_methods`, on remplace cette liste : ici `PUT` et `DELETE` ne sont plus retentées, et `POST` l'est. N'ajoutez `POST` que si l'API peut recevoir deux fois la même requête sans créer deux fois la ressource.

Avec `raise_on_status=False`, une fois les tentatives épuisées, on récupère la dernière réponse (une 503 par exemple) et c'est `raise_for_status()` qui lève l'exception. Sans cette option, requests lève directement une `requests.exceptions.RetryError`.

Le même `HTTPAdapter` gère le pool de connexions : `pool_connections` est le nombre de pools gardés en mémoire (un par hôte) et `pool_maxsize` le nombre de connexions conservées dans chaque pool, 10 par défaut pour les deux. Comme le précise la documentation d'urllib3, garder plus d'une connexion par hôte ne sert qu'avec plusieurs threads.

## Timeouts et proxies

Une session n'a pas de timeout par défaut, et il n'existe pas d'attribut pour lui en donner un : il faut passer `timeout` à chaque requête. Sans timeout, une requête vers un serveur qui ne répond plus peut rester bloquée indéfiniment.

```python
with requests.Session() as s:
    # Proxy (par ex: réseau d'entreprise)
    s.proxies.update({
        "http": "http://proxy.local:3128",
        "https": "http://proxy.local:3128",
    })

    # Timeout par appel (connect, read)
    r = s.get("https://example.com/slow", timeout=(3.05, 10))
    r.raise_for_status()
```

Le tuple `(3.05, 10)` laisse 3,05 secondes pour établir la connexion et 10 secondes au serveur pour répondre. La documentation de requests conseille un timeout de connexion un peu supérieur à un multiple de 3 secondes, la fenêtre de retransmission des paquets TCP par défaut. Le timeout de lecture est le temps d'attente maximum entre deux octets reçus du serveur, pas la durée totale de la requête.

Attention aux proxies définis sur la session : s'il existe des variables d'environnement `HTTP_PROXY` ou `HTTPS_PROXY`, elles passent devant `s.proxies` (la documentation de requests le signale, voir l'issue [#2018](https://github.com/psf/requests/issues/2018)). Pour être sûr du proxy utilisé, passez `proxies=` à chaque requête, ou demandez à la session d'ignorer l'environnement avec `s.trust_env = False`. Elle ignore alors aussi le fichier `.netrc` et la variable `REQUESTS_CA_BUNDLE`.

## Vérification des certificats SSL

Par défaut, requests vérifie le certificat des serveurs HTTPS (`verify=True`). On peut ajuster ce comportement au niveau de la session :

```python
with requests.Session() as s:
    # Utiliser un bundle CA personnalisé (ex: autorité interne d'entreprise)
    s.verify = "/chemin/vers/ca-bundle.pem"

    # Certificat client (mTLS) : un seul fichier (clé + certificat)
    # ou un tuple (certificat, clé)
    s.cert = ("/chemin/client.crt", "/chemin/client.key")

    r = s.get("https://api.interne.example.com/me", timeout=10)
    r.raise_for_status()
```

`verify` accepte `True` ou le chemin d'un bundle de certificats d'autorités de confiance (un fichier, ou un dossier préparé avec l'utilitaire `c_rehash` d'OpenSSL). `s.verify = False` désactive la vérification : n'importe quel certificat est alors accepté, ce qui expose les échanges à une attaque de type *man-in-the-middle*. À réserver au développement en local.

`cert` sert à présenter un certificat client, quand le serveur exige une authentification mutuelle (mTLS).

Comme pour les proxies, les variables d'environnement `REQUESTS_CA_BUNDLE` et `CURL_CA_BUNDLE` passent devant `s.verify` quand elles sont définies.

## Un client d'API réutilisable

Dans un vrai projet, on peut regrouper toute cette configuration dans une petite classe qui crée sa session, puis ajoute le timeout et le `raise_for_status()` à chaque appel :

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

L'annotation `token: str | None` demande Python 3.10 ou plus récent. Sur une version antérieure, utilisez `Optional[str]` (avec `from typing import Optional`).

Plutôt que de garder une session globale pour tout le programme, créez le client là où vous en avez besoin et passez-le aux fonctions qui l'utilisent. Si votre programme utilise des threads, prévoyez un client par thread : la documentation de requests ne garantit pas qu'une session soit *thread-safe*.

## Voir aussi

- [Python : Comment faire des requêtes HTTP avec requests]({% post_url 2020-05-22-Comment-faire-des-requetes-http-en-python-avec-requests %})
- [Python : Comment utiliser les différents modes d'authentification avec requests]({% post_url 2025-09-05-Comment-utiliser-l-authentification-avec-requests %})
- [Documentation de requests : Session Objects](https://requests.readthedocs.io/en/latest/user/advanced/#session-objects)
- [Documentation d'urllib3 : Retry](https://urllib3.readthedocs.io/en/stable/reference/urllib3.util.html#urllib3.util.Retry)
