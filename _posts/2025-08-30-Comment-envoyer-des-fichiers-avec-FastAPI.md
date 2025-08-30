---
layout: article
title: "Comment envoyer des fichiers avec FastAPI"
author: Pierre Chopinet
tags:
  - python
  - fastapi
  - api
  - http
  - tutoriel
  - upload
  - fichiers
---

Dans ce tutoriel, nous allons voir comment passer (uploader) des fichiers à une API web réalisée avec FastAPI : un fichier simple, plusieurs fichiers, des champs de formulaire additionnels, la sauvegarde sur disque et quelques validations utiles.
<!--more-->

Pré‑requis (recommandé) :

- [Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %})
- [Organiser une application FastAPI en plusieurs fichiers]({% post_url 2025-08-17-Organiser-une-application-FastAPI-en-plusieurs-fichiers %})
- [Comment dockeriser une application FastAPI]({% post_url 2025-08-16-Comment-dockeriser-une-api-web-avec-FastAPI %})
- Côté client : [Comment faire des requêtes HTTP en python avec requests]({% post_url 2020-05-22-Comment-faire-des-requetes-http-en-python-avec-requests %})

## Pourquoi "multipart/form-data" ?

L'envoi de fichiers via HTTP se fait classiquement avec le type de contenu `multipart/form-data`. C'est exactement ce que sait traiter FastAPI lorsqu'on déclare des paramètres de type `File(...)` et `UploadFile`.

- `bytes` + `File(...)` : lit tout le fichier en mémoire (simple mais à éviter pour les gros fichiers)
- `UploadFile` + `File(...)` : fournit un objet fichier en streaming (plus efficace), avec les métadonnées `filename` et `content_type`

## Un premier upload (fichier unique)

Créez un fichier `app.py` :

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

Lancement :

```bash
uvicorn app:app --reload
```

Test rapide avec `curl` :

```bash
curl -F "file=@monimage.png" http://127.0.0.1:8000/uploadfile
```

### Variante: lire en mémoire avec `bytes`

```python
from fastapi import FastAPI, File

app = FastAPI()

@app.post("/upload-bytes")
async def upload_bytes(file: bytes = File(...)):
    # file est un bytes qui contient tout le contenu
    return {"size": len(file)}
```

Avantage: simplicité. Inconvénient: le contenu est entièrement chargé en mémoire.

## Ajouter des champs de formulaire

Il est fréquent d'envoyer des métadonnées avec le fichier (ex: `user_id`, `tags`). Pour cela, combinez `File(...)` et `Form(...)` :

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

Côté `curl` :

```bash
curl -F "file=@report.pdf" -F "user_id=123" -F "description=rapport trimestriel" \
  http://127.0.0.1:8000/upload-with-meta
```

Avec `requests` en Python :

```python
import requests

url = "http://127.0.0.1:8000/upload-with-meta"
files = {"file": ("report.pdf", open("report.pdf", "rb"), "application/pdf")}
data = {"user_id": 123, "description": "rapport trimestriel"}
resp = requests.post(url, files=files, data=data)
print(resp.json())
```

## Uploader plusieurs fichiers

Pour accepter plusieurs fichiers, utilisez une liste d'`UploadFile` :

```python
from typing import List
from fastapi import FastAPI, UploadFile, File

app = FastAPI()

@app.post("/uploadfiles")
async def upload_files(files: List[UploadFile] = File(...)):
    return [{"filename": f.filename, "type": f.content_type} for f in files]
```

`curl` :

```bash
curl -F "files=@a.png" -F "files=@b.png" http://127.0.0.1:8000/uploadfiles
```

`requests` :

```python
import requests

url = "http://127.0.0.1:8000/uploadfiles"
files = [
    ("files", ("a.png", open("a.png", "rb"), "image/png")),
    ("files", ("b.png", open("b.png", "rb"), "image/png")),
]
resp = requests.post(url, files=files)
print(resp.json())
```

## Sauvegarder le fichier sur disque (streaming)

Avec `UploadFile`, vous pouvez copier le flux directement sans charger le fichier en mémoire :

```python
import os
import shutil
from fastapi import FastAPI, UploadFile, File

app = FastAPI()

@app.post("/uploadfile/save")
async def save_file(file: UploadFile = File(...)):
    os.makedirs("uploads", exist_ok=True)
    dest_path = os.path.join("uploads", file.filename)
    with open(dest_path, "wb") as out:
        shutil.copyfileobj(file.file, out)  # copie en streaming
    return {"saved_as": dest_path}
```

> Remarque : si vous avez structuré votre app avec des `routers` (voir l'article sur l'[organisation en plusieurs fichiers]({% post_url 2025-08-17-Organiser-une-application-FastAPI-en-plusieurs-fichiers %})), placez ces endpoints dans un module dédié (`routers/upload.py`) et incluez-le avec `include_router`.

## Valider le type et la taille

Exemple simple de validation du type MIME et d'une limite de taille (lecture en mémoire — pour de gros fichiers, préférez vérifier pendant la copie et interrompre au‑delà d'un seuil) :

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

> Astuce : pour traiter des CSV après upload, vous pouvez utiliser `pandas.read_csv`. Voir l'article : [Comment sauvegarder un dataframe pandas]({% post_url 2023-12-28-Comment-sauvegarder-un-dataframe-pandas %}).

## Exemple client complet (requests)

```python
import requests

base = "http://127.0.0.1:8000"

# 1) Fichier unique
with open("monimage.png", "rb") as f:
    resp = requests.post(f"{base}/uploadfile", files={"file": ("monimage.png", f, "image/png")})
    print(resp.json())

# 2) Plusieurs fichiers
files = [
    ("files", ("a.png", open("a.png", "rb"), "image/png")),
    ("files", ("b.png", open("b.png", "rb"), "image/png")),
]
resp = requests.post(f"{base}/uploadfiles", files=files)
print(resp.json())

# 3) Fichier + métadonnées
with open("report.pdf", "rb") as f:
    resp = requests.post(
        f"{base}/upload-with-meta",
        files={"file": ("report.pdf", f, "application/pdf")},
        data={"user_id": 123, "description": "rapport trimestriel"},
    )
    print(resp.json())
```

## Le code complet (exemple minimal)

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
async def save_file(file: UploadFile = File(...)):
    os.makedirs("uploads", exist_ok=True)
    dest_path = os.path.join("uploads", file.filename)
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

- [Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %})
- [Organiser une application FastAPI en plusieurs fichiers]({% post_url 2025-08-17-Organiser-une-application-FastAPI-en-plusieurs-fichiers %})
- [Comment dockeriser une application FastAPI]({% post_url 2025-08-16-Comment-dockeriser-une-api-web-avec-FastAPI %})
- [Comment faire des requêtes HTTP en python avec requests]({% post_url 2020-05-22-Comment-faire-des-requetes-http-en-python-avec-requests %})
- [Documentation FastAPI — Request Files](https://fastapi.tiangolo.com/tutorial/request-files/)
