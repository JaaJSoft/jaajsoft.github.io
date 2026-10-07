---
layout: article
title: "Python : Comment utiliser les différents modes d'authentification avec requests"
tags:
  - python
  - http
  - api
  - rest
  - requests
  - authentification
  - sécurité
  - oauth
author: Pierre Chopinet
---

Dans ce tutoriel, nous allons voir comment s'authentifier auprès d'une API avec la bibliothèque Python requests : Basic, Digest, token Bearer, clé d'API, OAuth1 et OAuth2 avec requests-oauthlib, ou encore un fichier `.netrc`.
<!--more-->

Dans cet article :
- Installation
- Basic Auth
- Digest Auth
- Bearer token
- Clé d'API
- OAuth1 avec requests-oauthlib
- OAuth2 avec requests-oauthlib
- Le fichier .netrc
- Écrire sa propre classe d'authentification
- Proxies avec authentification

## Installation

```bash
pip install requests
```

Pour OAuth (OAuth1 et OAuth2), il faut en plus requests-oauthlib :

```bash
pip install requests-oauthlib
```

Sous Windows, si la commande `pip` n'est pas reconnue, utilisez `python -m pip install ...`. Les exemples ont été testés avec requests 2.34.2 et requests-oauthlib 2.0.0 (avec oauthlib 4.0.0).

## Basic Auth

Le mode le plus simple : l'identifiant et le mot de passe sont envoyés dans l'en-tête `Authorization`, encodés en base64. Avec requests, il suffit de passer un tuple au paramètre `auth` :

```python
import requests

url = "https://api.example.com/ressource"
resp = requests.get(url, auth=("mon_user", "mon_mot_de_passe"))
resp.raise_for_status()
print(resp.json())
```

Le tuple est un raccourci pour la classe `HTTPBasicAuth` :

```python
from requests.auth import HTTPBasicAuth
resp = requests.get(url, auth=HTTPBasicAuth("mon_user", "mon_mot_de_passe"))
```

Avec une session, l'authentification est définie une fois pour toutes les requêtes (voir [l'article sur les sessions]({% post_url 2025-09-04-Comment-utiliser-les-sessions-avec-requests %})) :

```python
with requests.Session() as s:
    s.auth = ("mon_user", "mon_mot_de_passe")
    r1 = s.get("https://api.example.com/profile")
    r2 = s.get("https://api.example.com/orders")
```

requests n'attend pas que le serveur réponde 401 pour s'authentifier : l'en-tête part dès la première requête. Cet en-tête n'a rien de mystérieux, on peut le construire soi-même et on obtient exactement le même :

```python
import base64, requests
user, pwd = "mon_user", "mon_mot_de_passe"
token = base64.b64encode(f"{user}:{pwd}".encode()).decode()
headers = {"Authorization": f"Basic {token}"}
requests.get("https://api.example.com/", headers=headers)
```

Ici `token` vaut `bW9uX3VzZXI6bW9uX21vdF9kZV9wYXNzZQ==`, que n'importe qui peut décoder. Le base64 n'est pas un chiffrement : n'utilisez Basic Auth qu'en HTTPS.

Dans ces exemples, les identifiants sont écrits en dur pour rester lisibles. Dans un vrai programme, lisez-les plutôt depuis des variables d'environnement, par exemple avec [python-dotenv]({% post_url 2026-04-20-Comment-utiliser-les-variables-d-environnement-avec-python-dotenv %}).

## Digest Auth

Avec Digest, le mot de passe ne circule pas : le client envoie une empreinte calculée à partir du mot de passe et d'une valeur aléatoire fournie par le serveur. requests le gère nativement :

```python
from requests.auth import HTTPDigestAuth
import requests

resp = requests.get(
    "https://httpbin.org/digest-auth/auth/user/pass",
    auth=HTTPDigestAuth("user", "pass")
)
print(resp.status_code)
```

Le script affiche `200`. Contrairement à Basic, il faut deux allers-retours : la première requête reçoit une réponse 401 avec le *challenge* du serveur, puis requests la rejoue avec l'empreinte. Cette réponse 401 intermédiaire est visible dans `resp.history`.

## Bearer token

Beaucoup d'API récentes utilisent un jeton (*token*), souvent un JWT obtenu via OAuth2, à envoyer dans l'en-tête `Authorization: Bearer <token>` :

```python
import requests

token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
headers = {"Authorization": f"Bearer {token}"}
resp = requests.get("https://api.example.com/me", headers=headers, timeout=10)
resp.raise_for_status()
print(resp.json())
```

Avec une session :

```python
s = requests.Session()
s.headers.update({"Authorization": f"Bearer {token}"})
# toutes les requêtes de s incluent l'en-tête Authorization
```

Quand le token expire, il faut en obtenir un nouveau (avec le *refresh token* par exemple) et mettre à jour l'en-tête de la session. Avec OAuth2, `OAuth2Session` peut s'en charger, on le voit plus bas.

## Clé d'API

Certaines API utilisent une clé statique, à transmettre dans un en-tête ou dans l'URL. Le nom de l'en-tête ou du paramètre dépend de l'API.

En header :

```python
headers = {"X-API-Key": "ma_cle_api"}
requests.get("https://api.example.com/data", headers=headers)
```

En paramètre d'URL :

```python
params = {"api_key": "ma_cle_api"}
requests.get("https://api.example.com/data", params=params)
```

Préférez l'en-tête quand l'API le permet : une clé dans l'URL se retrouve dans les logs des serveurs et des proxys.

## OAuth1 avec requests-oauthlib

OAuth1 a notamment été utilisé par l'API de Twitter. Chaque requête est signée avec deux paires de clés : celle de l'application (*consumer key* et *consumer secret*) et celle de l'utilisateur (*access token* et *access token secret*).

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

Cet exemple est purement illustratif : l'API de Twitter/X est devenue payante en 2023 et cet endpoint n'est plus accessible librement. OAuth1 est d'ailleurs de moins en moins utilisé, la plupart des API sont passées à OAuth2.

## OAuth2 avec requests-oauthlib

OAuth2 est aujourd'hui le standard le plus courant pour sécuriser les API. Avec requests-oauthlib, on utilise `OAuth2Session`, une session requests qui sait obtenir le token, l'ajouter aux requêtes et le rafraîchir.

Attention, requests-oauthlib refuse les URL en HTTP simple (erreur `InsecureTransportError`). Pour tester avec un fournisseur qui tourne en local, définissez la variable d'environnement `OAUTHLIB_INSECURE_TRANSPORT=1`, mais jamais en production.

### Flux Client Credentials (machine à machine)

Ce flux sert quand il n'y a pas d'utilisateur final, pour une intégration de serveur à serveur. On l'indique à `OAuth2Session` en lui passant un `BackendApplicationClient` :

```python
from oauthlib.oauth2 import BackendApplicationClient
from requests_oauthlib import OAuth2Session

client_id = "CLIENT_ID"
client_secret = "CLIENT_SECRET"
token_url = "https://auth.example.com/oauth/token"

# Le client "backend" correspond au flux client_credentials
client = BackendApplicationClient(client_id=client_id)
oauth = OAuth2Session(client=client)

# Récupération du token
token = oauth.fetch_token(
    token_url=token_url,
    client_id=client_id,
    client_secret=client_secret,
)

# Appel API avec le Bearer token automatiquement géré
resp = oauth.get("https://api.example.com/me")
resp.raise_for_status()
print(resp.json())
```

Sans ce client, `OAuth2Session` part du principe qu'on utilise le flux Authorization Code, et `fetch_token()` lève `ValueError: Please supply either code or authorization_response parameters.`

Par défaut, `fetch_token()` envoie `client_id` et `client_secret` dans un en-tête Basic. Si votre fournisseur les attend dans le corps de la requête, ajoutez `include_client_id=True`.

### Flux Authorization Code (application web)

C'est le flux classique des applications web : on redirige l'utilisateur vers la page d'autorisation du fournisseur, qui le renvoie ensuite vers votre URL de *callback* avec un `code` à échanger contre un token.

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

Pour l'exemple, tout se passe dans le même script. Dans une vraie application web, la génération de l'URL et le *callback* sont deux requêtes HTTP différentes : gardez le `state` renvoyé par `authorization_url()` (dans la session de l'utilisateur par exemple), puis recréez la session dans la route de callback avec `OAuth2Session(client_id, redirect_uri=redirect_uri, state=state)`. `fetch_token()` vérifie que le `state` reçu est le bon, ce qui protège contre les attaques CSRF.

### Rafraîchir automatiquement le token

Quand le fournisseur renvoie aussi un `refresh_token`, `OAuth2Session` peut obtenir un nouveau token toute seule et vous le transmettre pour que vous le sauvegardiez :

```python
from requests_oauthlib import OAuth2Session

client_id = "CLIENT_ID"
client_secret = "CLIENT_SECRET"
token_url = "https://auth.example.com/oauth/token"

# token obtenu précédemment avec fetch_token() et sauvegardé tel quel
saved_token = {
    "access_token": "...",
    "refresh_token": "...",
    "token_type": "Bearer",
    "expires_in": 3600,
    "expires_at": 1767225600.0,  # ajouté par fetch_token(), à conserver
}

def save_token(token):
    # Persistant : fichier, base, vault
    print("Nouveau token sauvegardé")

extra = {"client_id": client_id, "client_secret": client_secret}

oauth = OAuth2Session(
    client_id=client_id,
    token=saved_token,
    auto_refresh_url=token_url,
    auto_refresh_kwargs=extra,
    token_updater=save_token,
)

# Si le token a expiré, OAuth2Session le rafraîchit avant d'envoyer la requête
resp = oauth.get("https://api.example.com/ressource")
print(resp.status_code)
```

Attention, requests-oauthlib sait qu'un token a expiré grâce à sa date d'expiration, pas grâce à la réponse du serveur : une réponse 401 ne déclenche pas de rafraîchissement. Le dictionnaire renvoyé par `fetch_token()` contient un champ `expires_at` (un timestamp), d'où l'intérêt de sauvegarder le token tel quel. Avec seulement `expires_in`, l'expiration est comptée à partir de la création de la session. Et si vous ne fournissez pas de `token_updater`, la session lève une exception `TokenUpdated` qui contient le nouveau token.

La liste des *scopes* dépend du fournisseur : lisez sa documentation. Enfin, stockez les tokens de manière sécurisée (base de données, fichier chiffré, *vault*) et évitez de les écrire dans les logs.

## Le fichier .netrc

Si vous ne passez pas de paramètre `auth`, requests cherche des identifiants pour l'hôte dans `~/.netrc` ou `~/_netrc`, ou dans le fichier indiqué par la variable d'environnement `NETRC`. Le `~` correspond à `$HOME` sous Linux et macOS, et à `%USERPROFILE%` sous Windows.

Contenu d'exemple :

```
machine api.example.com
  login mon_user
  password mon_mot_de_passe
```

Utilisation :

```python
import requests
# Pas de auth= ; requests va chercher dans ~/.netrc ou ~/_netrc
resp = requests.get("https://api.example.com/secure")
```

Les identifiants trouvés sont envoyés en Basic Auth. Attention, une entrée du `.netrc` remplace un en-tête `Authorization` passé avec `headers=`, un token Bearer par exemple : si une requête part avec les mauvais identifiants, pensez à vérifier ce fichier. Le paramètre `auth=` passe toujours devant le `.netrc`, et une session avec `trust_env = False` ne le lit pas.

Ce fichier contient des mots de passe en clair : sous Linux et macOS, limitez ses droits avec `chmod 600 ~/.netrc`, et sous Windows, restreignez les permissions de `_netrc` à votre compte.

## Écrire sa propre classe d'authentification

Pour un schéma d'authentification que requests ne connaît pas, on hérite de `requests.auth.AuthBase` et on modifie la requête dans la méthode `__call__` :

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

Cela fonctionne aussi avec une session :

```python
s = requests.Session()
s.auth = TokenAuth("mon_token")
```

## Proxies avec authentification

Pour un proxy qui demande une authentification, les identifiants se placent dans l'URL du proxy :

```python
proxies = {
    "http":  "http://user:pass@proxy.local:8080",
    "https": "http://user:pass@proxy.local:8080",
}
requests.get("https://api.example.com/", proxies=proxies, timeout=10)
```

requests les envoie au proxy dans l'en-tête `Proxy-Authorization`. On peut aussi passer par les variables d'environnement, que requests lit automatiquement. En PowerShell :

```powershell
$env:HTTPS_PROXY = "http://user:pass@proxy.local:8080"
python mon_script.py
```

En Bash :

```bash
export HTTPS_PROXY="http://user:pass@proxy.local:8080"
# Optionnellement aussi pour HTTP non chiffré
export HTTP_PROXY="http://user:pass@proxy.local:8080"
python mon_script.py
# Ou pour une seule commande sans polluer l'environnement global :
HTTPS_PROXY="http://user:pass@proxy.local:8080" python mon_script.py
```

## Voir aussi

- [Python : Comment faire des requêtes HTTP avec requests]({% post_url 2020-05-22-Comment-faire-des-requetes-http-en-python-avec-requests %})
- [Python : Comment utiliser les sessions avec requests]({% post_url 2025-09-04-Comment-utiliser-les-sessions-avec-requests %})
- [Python : Comment utiliser les variables d'environnement avec python-dotenv]({% post_url 2026-04-20-Comment-utiliser-les-variables-d-environnement-avec-python-dotenv %})
- [Documentation de requests : Authentication](https://requests.readthedocs.io/en/latest/user/authentication/)
- [Documentation de requests-oauthlib](https://requests-oauthlib.readthedocs.io/)
