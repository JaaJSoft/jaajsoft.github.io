---
layout: article
title: "Uploader des fichiers avec FastAPI"
description: "Uploader des fichiers avec FastAPI : fichier unique ou multiple, champs de formulaire, sauvegarde sur disque en streaming et validation du type et de la taille."
author: Pierre Chopinet
tags:
  - python
  - fastapi
  - api
  - rest
  - http
  - upload
  - fichiers
---

Dans ce tutoriel, nous allons voir comment envoyer des fichiers à une API web
FastAPI : un fichier seul, plusieurs fichiers, un fichier accompagné de champs
de formulaire, puis comment l'enregistrer sur le disque et vérifier son type et
sa taille.
<!--more-->

Dans cet article :
- Installation
- Le format multipart/form-data
- Un premier upload
- Ajouter des champs de formulaire
- Uploader plusieurs fichiers
- Sauvegarder le fichier sur disque
- Valider le type et la taille
- Le code complet

Pré-requis : côté serveur,
[Python : Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %}) ;
côté client,
[Python : Comment faire des requêtes HTTP avec requests]({% post_url 2020-05-22-Comment-faire-des-requetes-http-en-python-avec-requests %}).

## Installation

Pour lire les fichiers et les champs de formulaire, FastAPI a besoin de la
librairie `python-multipart`. On l'installe avec FastAPI et uvicorn :

```bash
pip install fastapi uvicorn python-multipart
```

Sans elle, l'application ne démarre même pas : dès qu'une route déclare un
paramètre `File()` ou `Form()`, FastAPI lève l'erreur
`RuntimeError: Form data requires "python-multipart" to be installed.`

## Le format multipart/form-data

Un navigateur qui envoie un formulaire contenant un fichier utilise le type de
contenu `multipart/form-data` : le corps de la requête est découpé en parties,
une par champ ou par fichier. C'est ce format que FastAPI attend quand on
déclare un paramètre avec `File()`.

On peut récupérer un fichier de deux façons :

- avec le type `bytes`, on reçoit directement tout le contenu du fichier en
  mémoire, ce qui convient aux petits fichiers ;
- avec le type `UploadFile`, on reçoit un objet qui donne le nom du fichier
  (`filename`), son type (`content_type`) et un fichier Python (`file`). Le
  contenu est gardé en mémoire jusqu'à 1 Mo, puis écrit dans un fichier
  temporaire sur le disque, ce qui permet de recevoir de gros fichiers sans
  remplir la RAM.

Dans les deux cas, FastAPI reçoit le fichier en entier avant d'appeler votre
fonction.

## Un premier upload

Créez un fichier `app.py` :

```python
from fastapi import FastAPI, UploadFile, File

app = FastAPI()

@app.post("/uploadfile")
async def upload_file(file: UploadFile = File(...)):
    # Si le fichier est petit et que vous avez besoin de sa taille
    content = await file.read()  # attention: lit tout en mémoire
    return {
        "filename": file.filename,
        "content_type": file.content_type,
        "size": len(content),
    }
```

On lance l'API :

```bash
uvicorn app:app --reload
```

Avec `curl`, l'option `-F` envoie un formulaire en `multipart/form-data`, et le
`@` indique un fichier à joindre :

```bash
curl -F "file=@monimage.png" http://127.0.0.1:8000/uploadfile
{"filename":"monimage.png","content_type":"image/png","size":55005}
```

Le nom du champ (`file`) doit correspondre au nom du paramètre de la fonction.

Même chose en Python avec `requests`, en passant le fichier dans `files` sous la
forme d'un tuple (nom, fichier ouvert, type) :

```python
import requests

with open("monimage.png", "rb") as f:
    resp = requests.post(
        "http://127.0.0.1:8000/uploadfile",
        files={"file": ("monimage.png", f, "image/png")},
    )
print(resp.json())
```

```
{'filename': 'monimage.png', 'content_type': 'image/png', 'size': 55005}
```

### Variante : lire le contenu avec `bytes`

```python
from fastapi import FastAPI, File

app = FastAPI()

@app.post("/upload-bytes")
async def upload_bytes(file: bytes = File(...)):
    # file est un bytes qui contient tout le contenu
    return {"size": len(file)}
```

C'est plus simple, mais tout le contenu est chargé en mémoire et on perd le nom
et le type du fichier.

## Ajouter des champs de formulaire

On envoie souvent des informations avec le fichier (un identifiant
d'utilisateur, une description...). On les déclare avec `Form()`, à côté du
`File()` :

```python
from fastapi import FastAPI, UploadFile, File, Form
from typing import Optional

app = FastAPI()

@app.post("/upload-with-meta")
async def upload_with_meta(
    file: UploadFile = File(...),
    user_id: int = Form(...),
    description: Optional[str] = Form(None),
):
    return {
        "filename": file.filename,
        "user_id": user_id,
        "description": description,
    }
```

Avec `curl`, chaque champ est un `-F` de plus :

```bash
curl -F "file=@report.pdf" -F "user_id=123" -F "description=rapport trimestriel" \
  http://127.0.0.1:8000/upload-with-meta
{"filename":"report.pdf","user_id":123,"description":"rapport trimestriel"}
```

Le champ `user_id` est obligatoire et converti en entier : s'il manque, FastAPI
répond avec une erreur 422.

Avec `requests`, les champs de formulaire passent dans `data` :

```python
import requests

url = "http://127.0.0.1:8000/upload-with-meta"
data = {"user_id": 123, "description": "rapport trimestriel"}
with open("report.pdf", "rb") as f:
    files = {"file": ("report.pdf", f, "application/pdf")}
    resp = requests.post(url, files=files, data=data)
print(resp.json())
```

```
{'filename': 'report.pdf', 'user_id': 123, 'description': 'rapport trimestriel'}
```

## Uploader plusieurs fichiers

Pour accepter plusieurs fichiers dans le même champ, on déclare une liste
d'`UploadFile` :

```python
from typing import List
from fastapi import FastAPI, UploadFile, File

app = FastAPI()

@app.post("/uploadfiles")
async def upload_files(files: List[UploadFile] = File(...)):
    return [{"filename": f.filename, "type": f.content_type} for f in files]
```

Côté `curl`, on répète le même nom de champ :

```bash
curl -F "files=@a.png" -F "files=@b.png" http://127.0.0.1:8000/uploadfiles
[{"filename":"a.png","type":"image/png"},{"filename":"b.png","type":"image/png"}]
```

Avec `requests`, `files` devient une liste de tuples qui portent tous le nom
`files`. Le `ExitStack` ouvre les fichiers et garantit qu'ils seront tous
fermés à la sortie du bloc `with` :

```python
import requests
from contextlib import ExitStack

url = "http://127.0.0.1:8000/uploadfiles"
paths = [("a.png", "image/png"), ("b.png", "image/png")]

with ExitStack() as stack:
    files = [
        ("files", (name, stack.enter_context(open(name, "rb")), content_type))
        for name, content_type in paths
    ]
    resp = requests.post(url, files=files)

print(resp.json())
```

```
[{'filename': 'a.png', 'type': 'image/png'}, {'filename': 'b.png', 'type': 'image/png'}]
```

## Sauvegarder le fichier sur disque

Avec `UploadFile`, on peut copier le fichier reçu vers sa destination par
morceaux, sans jamais le charger entièrement en mémoire :

```python
import os
import shutil
from fastapi import FastAPI, UploadFile, File

app = FastAPI()

@app.post("/uploadfile/save")
def save_file(file: UploadFile = File(...)):
    os.makedirs("uploads", exist_ok=True)
    # On assainit le nom fourni par le client pour éviter un "path traversal" :
    # un nom du type "../../etc/passwd" pourrait sinon écrire hors du dossier uploads.
    safe_name = os.path.basename(file.filename)
    dest_path = os.path.join("uploads", safe_name)
    with open(dest_path, "wb") as out:
        shutil.copyfileobj(file.file, out)  # copie par morceaux
    return {"saved_as": dest_path}
```

```bash
curl -F "file=@monimage.png" http://127.0.0.1:8000/uploadfile/save
{"saved_as":"uploads/monimage.png"}
```

La fonction est déclarée avec `def` et pas `async def` : `open()` et
`shutil.copyfileobj()` sont des opérations bloquantes, et FastAPI exécute les
fonctions `def` dans un thread à part, ce qui évite de bloquer les autres
requêtes pendant la copie.

Le `os.path.basename()` n'est pas là pour faire joli : le nom du fichier est
choisi par le client, et FastAPI le transmet tel quel. Un client peut donc
envoyer un nom comme `../../evil.png`, que `basename()` ramène à `evil.png` :

```bash
curl -F "file=@monimage.png;filename=../../evil.png" http://127.0.0.1:8000/uploadfile/save
{"saved_as":"uploads/evil.png"}
```

Si votre application est découpée en plusieurs fichiers, ces routes ont leur
place dans un module dédié (`routers/upload.py`) inclus avec `include_router`,
voir
[Organiser une application FastAPI en plusieurs fichiers]({% post_url 2025-08-17-Organiser-une-application-FastAPI-en-plusieurs-fichiers %}).

## Valider le type et la taille

Voici un exemple simple qui vérifie le type MIME du fichier et refuse les
fichiers de plus de 10 Mo :

```python
from fastapi import FastAPI, UploadFile, File, HTTPException

app = FastAPI()

ALLOWED_TYPES = {"image/png", "image/jpeg", "text/csv", "application/pdf"}
MAX_SIZE = 10 * 1024 * 1024  # 10 Mo

@app.post("/uploadfile/validate")
async def upload_validate(file: UploadFile = File(...)):
    if file.content_type not in ALLOWED_TYPES:
        raise HTTPException(status_code=400, detail=f"Type non autorisé: {file.content_type}")

    content = await file.read()
    if len(content) > MAX_SIZE:
        raise HTTPException(status_code=413, detail="Fichier trop volumineux")

    # Réutilisation du flux après read()
    await file.seek(0)

    return {"filename": file.filename, "size": len(content)}
```

Le code 413 signifie que le contenu envoyé est trop gros. Le `seek(0)` remet le
curseur au début du fichier, pour pouvoir le relire ensuite (pour le sauvegarder
par exemple). Ici, la taille est mesurée en lisant tout le fichier en mémoire.
Pour de gros fichiers, on peut s'en passer : `UploadFile` donne aussi la taille
du fichier reçu dans `file.size`, sans rien lire.

Attention, `content_type` est le type déclaré par le client, pas le résultat
d'une analyse du fichier : rien n'empêche d'envoyer un script en le déclarant
comme une image.

```bash
curl -F "file=@app.py;type=image/png" http://127.0.0.1:8000/uploadfile/validate
{"filename":"app.py","size":1855}
```

Inversement, `curl` ne devine le type qu'à partir de l'extension, et seulement
pour quelques formats courants (png, jpg, pdf...). Un fichier `.csv` part en
`application/octet-stream` et se fait refuser, il faut préciser son type :

```bash
curl -F "file=@ventes.csv" http://127.0.0.1:8000/uploadfile/validate
{"detail":"Type non autorisé: application/octet-stream"}

curl -F "file=@ventes.csv;type=text/csv" http://127.0.0.1:8000/uploadfile/validate
{"filename":"ventes.csv","size":33}
```

Une fois le fichier accepté, un CSV peut être chargé directement avec
`pandas.read_csv(file.file)` (voir l'article sur
[la sauvegarde et le chargement de dataframes Pandas]({% post_url 2023-12-28-Comment-sauvegarder-un-dataframe-pandas %})).

## Le code complet

```python
from typing import List, Optional
from fastapi import FastAPI, UploadFile, File, Form, HTTPException
import os, shutil

app = FastAPI()

@app.get("/")
def root():
    return "Hello World"

@app.post("/uploadfile")
async def upload_file(file: UploadFile = File(...)):
    content = await file.read()
    return {"filename": file.filename, "content_type": file.content_type, "size": len(content)}

@app.post("/upload-bytes")
async def upload_bytes(file: bytes = File(...)):
    return {"size": len(file)}

@app.post("/upload-with-meta")
async def upload_with_meta(
    file: UploadFile = File(...),
    user_id: int = Form(...),
    description: Optional[str] = Form(None),
):
    return {"filename": file.filename, "user_id": user_id, "description": description}

@app.post("/uploadfiles")
async def upload_files(files: List[UploadFile] = File(...)):
    return [{"filename": f.filename, "type": f.content_type} for f in files]

@app.post("/uploadfile/save")
def save_file(file: UploadFile = File(...)):
    os.makedirs("uploads", exist_ok=True)
    safe_name = os.path.basename(file.filename)  # évite le path traversal
    dest_path = os.path.join("uploads", safe_name)
    with open(dest_path, "wb") as out:
        shutil.copyfileobj(file.file, out)
    return {"saved_as": dest_path}

ALLOWED_TYPES = {"image/png", "image/jpeg", "text/csv", "application/pdf"}
MAX_SIZE = 10 * 1024 * 1024

@app.post("/uploadfile/validate")
async def upload_validate(file: UploadFile = File(...)):
    if file.content_type not in ALLOWED_TYPES:
        raise HTTPException(status_code=400, detail=f"Type non autorisé: {file.content_type}")
    content = await file.read()
    if len(content) > MAX_SIZE:
        raise HTTPException(status_code=413, detail="Fichier trop volumineux")
    await file.seek(0)
    return {"filename": file.filename, "size": len(content)}
```

## Voir aussi

- [Python : Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %})
- [Organiser une application FastAPI en plusieurs fichiers]({% post_url 2025-08-17-Organiser-une-application-FastAPI-en-plusieurs-fichiers %})
- [Python : Comment faire des requêtes HTTP avec requests]({% post_url 2020-05-22-Comment-faire-des-requetes-http-en-python-avec-requests %})
- [Request Files](https://fastapi.tiangolo.com/tutorial/request-files/), la documentation de FastAPI sur l'upload de fichiers
