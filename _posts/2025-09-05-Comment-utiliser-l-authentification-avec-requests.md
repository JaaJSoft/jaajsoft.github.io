---
layout: article
title: "Python : Comment utiliser les différents modes d'authentification avec requests"
author: Pierre Chopinet
tags:
  - python
  - http
  - requests
  - authentification
  - sécurité
  - oauth
  - tutoriel
---

Dans cet article, on passe en revue les principaux modes d’authentification supportés par la bibliothèque Python requests, avec des exemples concrets et des bonnes pratiques (sessions, retries, proxies, sécurité).
<!--more-->

## Installation rapide

```bash
pip install requests
```

Pour OAuth (OAuth1 et OAuth2) :

```bash
pip install requests-oauthlib
```

Sous Windows (PowerShell) :

```powershell
python -m pip install requests
python -m pip install requests-oauthlib
```

---

## 1) Basic Auth (HTTP Basic)

La manière la plus simple : envoyer identifiant/mot de passe encodés en base64 dans l’en-tête `Authorization`.

```python
import requests

url = "https://api.example.com/ressource"
resp = requests.get(url, auth=("mon_user", "mon_mot_de_passe"))
resp.raise_for_status()
print(resp.json())
```

Équivalent explicite :

```python
from requests.auth import HTTPBasicAuth
resp = requests.get(url, auth=HTTPBasicAuth("mon_user", "mon_mot_de_passe"))
```

Avec une Session (réutilise la connexion et l’auth sur plusieurs requêtes) :

```python
with requests.Session() as s:
    s.auth = ("mon_user", "mon_mot_de_passe")
    r1 = s.get("https://api.example.com/profile")
    r2 = s.get("https://api.example.com/orders")
```

Astuce (pré-authentification) : certains serveurs n’envoient le challenge 401 qu’après une 1re requête. Pour éviter un aller‑retour, on peut ajouter l’en-tête soi‑même :

```python
import base64, requests
user, pwd = "mon_user", "mon_mot_de_passe"
token = base64.b64encode(f"{user}:{pwd}".encode()).decode()
headers = {"Authorization": f"Basic {token}"}
requests.get("https://api.example.com/", headers=headers)
```

> Note : n’envoyez jamais des identifiants en clair en HTTP ; utilisez HTTPS.

---

## 2) Digest Auth (HTTP Digest)

Plus sûr que Basic (le mot de passe n’est pas envoyé tel quel), et aussi supporté nativement :

```python
from requests.auth import HTTPDigestAuth
import requests

resp = requests.get(
    "https://httpbin.org/digest-auth/auth/user/pass",
    auth=HTTPDigestAuth("user", "pass")
)
print(resp.status_code)
```

---

## 3) Bearer Token (OAuth2/JWT)

Beaucoup d’APIs modernes utilisent un jeton (token) dans l’en-tête `Authorization: Bearer <token>`.

```python
import requests

token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
headers = {"Authorization": f"Bearer {token}"}
resp = requests.get("https://api.example.com/me", headers=headers, timeout=10)
resp.raise_for_status()
print(resp.json())
```

Avec Session :

```python
s = requests.Session()
s.headers.update({"Authorization": f"Bearer {token}"})
# toutes les requêtes de s incluent l’en-tête Authorization
```

Si le token expire, prévoyez un rafraîchissement (endpoint refresh_token) et mettez à jour l’en‑tête dans la Session.

---

## 4) API Key (en header ou en paramètre de requête)

Certaines APIs utilisent une clé statique à transmettre en header ou en query string.

- En header :

```python
headers = {"X-API-Key": "ma_cle_api"}
requests.get("https://api.example.com/data", headers=headers)
```

- En paramètre d’URL :

```python
params = {"api_key": "ma_cle_api"}
requests.get("https://api.example.com/data", params=params)
```

Privilégiez le header quand c’est possible (logguer la clé dans l'URL n'est pas une très bonne idée).

---

## 5) OAuth1 (via requests-oauthlib)

Utilisé anciennement par exemple par l'API de Twitter.

```python
import requests
from requests_oauthlib import OAuth1

auth = OAuth1(
    client_key="CONSUMER_KEY",
    client_secret="CONSUMER_SECRET",
    resource_owner_key="ACCESS_TOKEN",
    resource_owner_secret="ACCESS_TOKEN_SECRET",
)

resp = requests.get("https://api.twitter.com/1.1/account/verify_credentials.json", auth=auth)
print(resp.status_code)
```
Ce type d'authentification est moins courant aujourd’hui, généralement remplacé par OAuth2.

## OAuth2 (via requests-oauthlib)

OAuth2 est aujourd’hui le standard le plus courant pour sécuriser les APIs. Avec `requests-oauthlib`, on utilise `OAuth2Session` pour obtenir le jeton (token), l’envoyer, et éventuellement le rafraîchir.

### a) Flux Client Credentials (machine-to-machine)

Quand il n’y a pas d’utilisateur final (intégrations serveur ↔ serveur).

```python
from requests_oauthlib import OAuth2Session

client_id = "CLIENT_ID"
client_secret = "CLIENT_SECRET"
token_url = "https://auth.example.com/oauth/token"

# On crée une session sans token initial
oauth = OAuth2Session(client_id=client_id)

# Récupération du token via le flux client_credentials
# Selon le provider, il faut passer client_id/secret dans auth=HTTPBasicAuth(...)
# ou dans la donnée POST (client_secret). Ici on les passe en argument.
token = oauth.fetch_token(
    token_url=token_url,
    client_id=client_id,
    client_secret=client_secret,
    include_client_id=True,
)

# Appel API avec le Bearer token automatiquement géré
resp = oauth.get("https://api.example.com/me")
resp.raise_for_status()
print(resp.json())
```

### b) Flux Authorization Code (application web / utilisateur)

Classique pour les applis web : on redirige l’utilisateur vers la page d’auth, puis on reçoit un `code` sur l’URL de callback, et on l’échange contre un token.

```python
from requests_oauthlib import OAuth2Session

client_id = "CLIENT_ID"
client_secret = "CLIENT_SECRET"
authorization_base_url = "https://auth.example.com/oauth/authorize"
token_url = "https://auth.example.com/oauth/token"
redirect_uri = "https://monapp.example.com/callback"
scope = ["read", "write"]

# Étape 1 : obtenir l'URL d'autorisation et le state
oauth = OAuth2Session(client_id, redirect_uri=redirect_uri, scope=scope)
authorization_url, state = oauth.authorization_url(authorization_base_url)
print("Allez sur cette URL et autorisez l'application:", authorization_url)

# Étape 2 : après redirection, votre route /callback reçoit l'URL complète
# Par exemple en dev, on la colle ici manuellement :
redirect_response = input("Collez l'URL de redirection complète: ")

# Étape 3 : échanger le code contre un token
token = oauth.fetch_token(
    token_url=token_url,
    authorization_response=redirect_response,
    client_secret=client_secret,
)

# Étape 4 : appeler l'API
resp = oauth.get("https://api.example.com/me")
print(resp.json())
```

### c) Rafraîchir automatiquement le token

Quand le provider vous donne un `refresh_token`, configurez la session pour qu’elle rafraîchisse automatiquement et sauvegarde le nouveau token.

```python
from requests_oauthlib import OAuth2Session

client_id = "CLIENT_ID"
client_secret = "CLIENT_SECRET"
token_url = "https://auth.example.com/oauth/token"

# token obtenu précédemment et contenant un refresh_token
saved_token = {
    "access_token": "...",
    "refresh_token": "...",
    "token_type": "Bearer",
    "expires_in": 3600,
}

def save_token(token):
    # Persistant : fichier, base, vault…
    print("Nouveau token sauvegardé")

extra = {"client_id": client_id, "client_secret": client_secret}

oauth = OAuth2Session(
    client_id=client_id,
    token=saved_token,
    auto_refresh_url=token_url,
    auto_refresh_kwargs=extra,
    token_updater=save_token,
)

# Si 401/expiration, OAuth2Session tentera un refresh et rejouera la requête
resp = oauth.get("https://api.example.com/ressource")
print(resp.status_code)
```

Notes :
- La liste des `scopes` dépend du provider (lisez sa doc).
- Stockez les tokens de manière sécurisée (chiffrement, variables d’environnement, vault).
- Évitez de logger les tokens.
- Selon les providers, le passage de client_id/secret se fait en Basic Auth ou dans le corps POST.

---

## 6) Utiliser un fichier .netrc / _netrc

requests sait lire automatiquement les identifiants depuis `~/.netrc` (Linux/macOS) ou `~/_netrc` (Windows) si vous n’indiquez pas `auth=` et que le fichier contient une entrée pour l’hôte.

Contenu d’exemple :

```
machine api.example.com
  login mon_user
  password mon_mot_de_passe
```

Utilisation :

```python
import requests
# Pas de auth= ; requests va chercher dans ~/.netrc ou ~/_netrc
resp = requests.get("https://api.example.com/secure")
```

Sur Windows, assurez des permissions restreintes au fichier `_netrc`.

---

## 7) Écrire sa propre classe d’authentification

Vous pouvez personnaliser l’auth en héritant de `requests.auth.AuthBase`.

```python
import requests
from requests.auth import AuthBase

class TokenAuth(AuthBase):
    def __init__(self, token: str):
        self.token = token
    def __call__(self, r):
        r.headers["Authorization"] = f"Bearer {self.token}"
        return r

resp = requests.get("https://api.example.com/", auth=TokenAuth("mon_token"))
```

Cela fonctionne aussi avec `Session` :

```python
s = requests.Session()
s.auth = TokenAuth("mon_token")
```

---

## 8) Proxies avec authentification

Deux façons courantes :

- Dans l’URL du proxy :

```python
proxies = {
    "http":  "http://user:pass@proxy.local:8080",
    "https": "http://user:pass@proxy.local:8080",
}
requests.get("https://api.example.com/", proxies=proxies, timeout=10)
```

- Via variables d’environnement powershell (HTTP(S)_PROXY) – pratique en ligne de commande :

```powershell
$env:HTTPS_PROXY = "http://user:pass@proxy.local:8080"
python mon_script.py
```

- En Bash :

```bash
export HTTPS_PROXY="http://user:pass@proxy.local:8080"
# Optionnellement aussi pour HTTP non chiffré
export HTTP_PROXY="http://user:pass@proxy.local:8080"
python mon_script.py
# Ou pour une seule commande sans polluer l'environnement global :
HTTPS_PROXY="http://user:pass@proxy.local:8080" python mon_script.py
```

---

## Pour conclure

- Basic : `auth=(user, pass)` ou `HTTPBasicAuth`.
- Digest : `HTTPDigestAuth`.
- Bearer : en‑tête `Authorization: Bearer <token>` .
- OAuth1 : `requests-oauthlib` + `OAuth1`.
- OAuth2 : `requests-oauthlib` + `OAuth2Session` (client_credentials, authorization_code, refresh_token).
- API Key : header (préféré) ou query param.
- Custom : classe `AuthBase`.

---

## Voir aussi

- [Python : Comment faire des requêtes HTTP avec requests]({% post_url 2020-05-22-Comment-faire-des-requetes-http-en-python-avec-requests %})
- [Python : Comment utiliser les sessions avec requests pour optimiser vos appels HTTP]({% post_url 2025-09-04-Comment-utiliser-les-sessions-avec-requests %})
- [Python : Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %})
- [Organiser une application FastAPI en plusieurs fichiers]({% post_url 2025-08-17-Organiser-une-application-FastAPI-en-plusieurs-fichiers %})
- Documentation officielle requests : [https://requests.readthedocs.io](https://requests.readthedocs.io)
