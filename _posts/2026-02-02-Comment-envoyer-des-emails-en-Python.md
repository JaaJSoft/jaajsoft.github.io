---
layout: article
title: "Python : Comment envoyer des emails"
author: Pierre Chopinet
tags:
  - python
  - email
  - smtp
---

Envoyer des emails en Python est simple grâce aux modules natifs `smtplib` et `email`. Que ce soit pour des notifications automatiques, des rapports ou des alertes, Python offre une API complète pour gérer l'envoi d'emails (texte, HTML, pièces jointes).
<!--more-->

Dans ce tutoriel, vous découvrirez :
- Comment envoyer un email simple avec `smtplib`
- Configurer Gmail, Outlook, et serveurs SMTP personnalisés
- Envoyer des emails HTML avec mise en forme
- Ajouter des pièces jointes (PDF, images, fichiers)
- Gérer les erreurs et les bonnes pratiques de sécurité
- Intégration avec Flask (Flask-Mail)

---

## 1) Installation et prérequis

Les modules `smtplib` et `email` sont **natifs** en Python (aucune installation requise).

```bash
python --version
# Python 3.7+ recommandé
```

---

## 2) Envoyer un email simple (texte brut)

### Code minimal

```python
import smtplib
from email.message import EmailMessage

# Créer le message
msg = EmailMessage()
msg['Subject'] = 'Test depuis Python'
msg['From'] = 'votre.email@gmail.com'
msg['To'] = 'destinataire@example.com'
msg.set_content('Ceci est un email de test envoyé depuis Python.')

# Envoyer via SMTP
with smtplib.SMTP('smtp.gmail.com', 587) as smtp:
    smtp.starttls()  # Connexion sécurisée
    smtp.login('votre.email@gmail.com', 'votre_mot_de_passe')
    smtp.send_message(msg)
    print("Email envoyé avec succès !")
```

**Explication** :
- `EmailMessage()` : crée un message email
- `msg['Subject']`, `msg['From']`, `msg['To']` : en-têtes
- `set_content()` : corps du message en texte brut
- `SMTP('smtp.gmail.com', 587)` : serveur Gmail sur le port 587 (STARTTLS)
- `starttls()` : upgrade vers une connexion chiffrée TLS
- `login()` : authentification
- `send_message()` : envoi du message

---

## 3) Configuration des serveurs SMTP populaires

### Gmail

**Prérequis** : activer les "Mots de passe d'application" (App Passwords)

1. Aller dans [Compte Google > Sécurité](https://myaccount.google.com/security)
2. Activer la validation en deux étapes
3. Générer un "Mot de passe d'application"

```python
SMTP_SERVER = 'smtp.gmail.com'
SMTP_PORT = 587
EMAIL = 'votre.email@gmail.com'
PASSWORD = 'abcd efgh ijkl mnop'  # Mot de passe d'application
```

### Outlook / Hotmail

```python
SMTP_SERVER = 'smtp-mail.outlook.com'
SMTP_PORT = 587
EMAIL = 'votre.email@outlook.com'
PASSWORD = 'votre_mot_de_passe'
```

### Office 365

```python
SMTP_SERVER = 'smtp.office365.com'
SMTP_PORT = 587
EMAIL = 'votre.email@entreprise.com'
PASSWORD = 'votre_mot_de_passe'
```

### Yahoo Mail

```python
SMTP_SERVER = 'smtp.mail.yahoo.com'
SMTP_PORT = 587
EMAIL = 'votre.email@yahoo.com'
PASSWORD = 'mot_de_passe_application'  # Générer sur Yahoo
```

### Serveur SMTP personnalisé

```python
SMTP_SERVER = 'mail.mondomaine.com'
SMTP_PORT = 587  # ou 465 pour SSL direct
EMAIL = 'contact@mondomaine.com'
PASSWORD = 'mot_de_passe'
```

---

## 4) Envoyer un email HTML

### Avec mise en forme

```python
import smtplib
from email.message import EmailMessage

msg = EmailMessage()
msg['Subject'] = 'Rapport mensuel'
msg['From'] = 'votre.email@gmail.com'
msg['To'] = 'destinataire@example.com'

# Contenu HTML
html_content = """
<html>
  <head></head>
  <body>
    <h1 style="color: #2e6c80;">Rapport du mois</h1>
    <p>Bonjour,</p>
    <p>Voici le <strong>rapport mensuel</strong> :</p>
    <ul>
      <li>Ventes : +15%</li>
      <li>Utilisateurs : 1 250</li>
      <li>Revenus : 50 000€</li>
    </ul>
    <p>Cordialement,<br>L'équipe</p>
  </body>
</html>
"""

msg.set_content('Version texte brut (fallback)')  # Fallback pour clients sans HTML
msg.add_alternative(html_content, subtype='html')

# Envoi
with smtplib.SMTP('smtp.gmail.com', 587) as smtp:
    smtp.starttls()
    smtp.login('votre.email@gmail.com', 'votre_mot_de_passe')
    smtp.send_message(msg)
    print("Email HTML envoyé !")
```

**Points clés** :
- `set_content()` : version texte (fallback)
- `add_alternative(..., subtype='html')` : version HTML
- Les clients email afficheront le HTML, sinon le texte brut

---

## 5) Envoyer à plusieurs destinataires

### Destinataires multiples (To, Cc, Bcc)

```python
msg = EmailMessage()
msg['Subject'] = 'Réunion d\'équipe'
msg['From'] = 'votre.email@gmail.com'
msg['To'] = 'alice@example.com, bob@example.com'  # Liste séparée par virgules
msg['Cc'] = 'manager@example.com'  # Copie
msg['Bcc'] = 'archive@example.com'  # Copie cachée

msg.set_content('Rappel : réunion demain à 10h.')

with smtplib.SMTP('smtp.gmail.com', 587) as smtp:
    smtp.starttls()
    smtp.login('votre.email@gmail.com', 'votre_mot_de_passe')
    smtp.send_message(msg)
```

**Alternative avec une liste** :

```python
destinataires = ['alice@example.com', 'bob@example.com', 'charlie@example.com']
msg['To'] = ', '.join(destinataires)
```

---

## 6) Ajouter des pièces jointes

### Fichier texte, PDF, image

```python
import smtplib
from email.message import EmailMessage
from pathlib import Path

msg = EmailMessage()
msg['Subject'] = 'Document joint'
msg['From'] = 'votre.email@gmail.com'
msg['To'] = 'destinataire@example.com'
msg.set_content('Veuillez trouver ci-joint le document.')

# Ajouter une pièce jointe
file_path = Path('rapport.pdf')
with open(file_path, 'rb') as f:
    file_data = f.read()
    file_name = file_path.name

msg.add_attachment(file_data, maintype='application', subtype='pdf', filename=file_name)

# Envoi
with smtplib.SMTP('smtp.gmail.com', 587) as smtp:
    smtp.starttls()
    smtp.login('votre.email@gmail.com', 'votre_mot_de_passe')
    smtp.send_message(msg)
    print(f"Email avec {file_name} envoyé !")
```

### Plusieurs pièces jointes

```python
fichiers = ['rapport.pdf', 'graphique.png', 'data.csv']

for fichier in fichiers:
    with open(fichier, 'rb') as f:
        file_data = f.read()
        file_name = Path(fichier).name

        # Détection automatique du type MIME
        import mimetypes
        mime_type, _ = mimetypes.guess_type(fichier)
        maintype, subtype = mime_type.split('/') if mime_type else ('application', 'octet-stream')

        msg.add_attachment(file_data, maintype=maintype, subtype=subtype, filename=file_name)
```

### Image inline (intégrée dans le HTML)

```python
msg = EmailMessage()
msg['Subject'] = 'Newsletter'
msg['From'] = 'votre.email@gmail.com'
msg['To'] = 'destinataire@example.com'

html = """
<html>
  <body>
    <h1>Nouvelle version disponible !</h1>
    <img src="cid:logo">
  </body>
</html>
"""

msg.add_alternative(html, subtype='html')

# Ajouter l'image avec un Content-ID
with open('logo.png', 'rb') as img:
    msg.get_payload()[0].add_related(img.read(), maintype='image', subtype='png', cid='<logo>')

# Envoi...
```

---

## 7) Gestion des erreurs

### Erreurs courantes

```python
import smtplib
from email.message import EmailMessage

try:
    msg = EmailMessage()
    msg['Subject'] = 'Test'
    msg['From'] = 'votre.email@gmail.com'
    msg['To'] = 'destinataire@example.com'
    msg.set_content('Test')

    with smtplib.SMTP('smtp.gmail.com', 587, timeout=10) as smtp:
        smtp.starttls()
        smtp.login('votre.email@gmail.com', 'votre_mot_de_passe')
        smtp.send_message(msg)
        print("✅ Email envoyé avec succès")

except smtplib.SMTPAuthenticationError:
    print("❌ Erreur d'authentification (email/mot de passe incorrect)")
except smtplib.SMTPException as e:
    print(f"❌ Erreur SMTP : {e}")
except Exception as e:
    print(f"❌ Erreur : {e}")
```

**Erreurs fréquentes** :
- `SMTPAuthenticationError` : identifiants incorrects
- `SMTPRecipientsRefused` : email destinataire invalide
- `SMTPServerDisconnected` : connexion perdue
- `socket.gaierror` : serveur SMTP introuvable

---

## 8) Utiliser SSL (port 465) au lieu de STARTTLS

### Connexion SSL directe

```python
import smtplib
from email.message import EmailMessage

msg = EmailMessage()
msg['Subject'] = 'Test SSL'
msg['From'] = 'votre.email@gmail.com'
msg['To'] = 'destinataire@example.com'
msg.set_content('Test avec SSL')

# SMTP_SSL sur le port 465
with smtplib.SMTP_SSL('smtp.gmail.com', 465) as smtp:
    smtp.login('votre.email@gmail.com', 'votre_mot_de_passe')
    smtp.send_message(msg)
    print("Email envoyé via SSL")
```

**Différence** :
- **Port 587 + STARTTLS** : connexion non chiffrée upgradée vers TLS (recommandé)
- **Port 465 + SSL** : connexion chiffrée dès le début (ancien standard, toujours supporté)

---

## 9) Sécurité : ne pas hardcoder les mots de passe

### Utiliser des variables d'environnement

```python
import os
from dotenv import load_dotenv

# Charger depuis .env
load_dotenv()

EMAIL = os.getenv('EMAIL')
PASSWORD = os.getenv('EMAIL_PASSWORD')

# Utilisation
smtp.login(EMAIL, PASSWORD)
```

**Fichier `.env`** :
```
EMAIL=votre.email@gmail.com
EMAIL_PASSWORD=abcd efgh ijkl mnop
```

**Installation** :
```bash
pip install python-dotenv
```

**⚠️ Important** : ajoutez `.env` à votre `.gitignore` pour ne pas committer vos secrets.

---

## 10) Intégration avec Flask (Flask-Mail)

Pour envoyer des emails dans une application web Flask.

### Installation

```bash
pip install Flask-Mail
```

### Configuration

```python
from flask import Flask
from flask_mail import Mail, Message

app = Flask(__name__)

# Configuration SMTP
app.config['MAIL_SERVER'] = 'smtp.gmail.com'
app.config['MAIL_PORT'] = 587
app.config['MAIL_USE_TLS'] = True
app.config['MAIL_USERNAME'] = 'votre.email@gmail.com'
app.config['MAIL_PASSWORD'] = 'votre_mot_de_passe'
app.config['MAIL_DEFAULT_SENDER'] = 'votre.email@gmail.com'

mail = Mail(app)

@app.route('/send')
def send_email():
    msg = Message('Hello from Flask', recipients=['destinataire@example.com'])
    msg.body = 'Ceci est un email envoyé depuis Flask.'
    msg.html = '<h1>Hello</h1><p>Email HTML depuis Flask.</p>'
    mail.send(msg)
    return 'Email envoyé !'

if __name__ == '__main__':
    app.run(debug=True)
```

---

## 11) Envoi asynchrone avec threading

Pour ne pas bloquer l'exécution lors de l'envoi d'emails.

```python
import smtplib
from email.message import EmailMessage
import threading

def envoyer_email_async(destinataire, sujet, contenu):
    def _send():
        msg = EmailMessage()
        msg['Subject'] = sujet
        msg['From'] = 'votre.email@gmail.com'
        msg['To'] = destinataire
        msg.set_content(contenu)

        with smtplib.SMTP('smtp.gmail.com', 587) as smtp:
            smtp.starttls()
            smtp.login('votre.email@gmail.com', 'votre_mot_de_passe')
            smtp.send_message(msg)
            print(f"Email envoyé à {destinataire}")

    thread = threading.Thread(target=_send)
    thread.start()

# Utilisation
envoyer_email_async('destinataire@example.com', 'Test async', 'Message de test')
print("L'envoi est en cours en arrière-plan...")
```

**Alternative avec asyncio** (Python 3.7+) :

```python
import asyncio
import smtplib
from email.message import EmailMessage

async def envoyer_email_async(destinataire, sujet, contenu):
    loop = asyncio.get_event_loop()
    await loop.run_in_executor(None, envoyer_email_sync, destinataire, sujet, contenu)

def envoyer_email_sync(destinataire, sujet, contenu):
    msg = EmailMessage()
    msg['Subject'] = sujet
    msg['From'] = 'votre.email@gmail.com'
    msg['To'] = destinataire
    msg.set_content(contenu)

    with smtplib.SMTP('smtp.gmail.com', 587) as smtp:
        smtp.starttls()
        smtp.login('votre.email@gmail.com', 'votre_mot_de_passe')
        smtp.send_message(msg)

# Utilisation
asyncio.run(envoyer_email_async('destinataire@example.com', 'Test', 'Message'))
```

---

## 12) Templates d'emails avec Jinja2

Pour générer des emails HTML dynamiques.

### Installation

```bash
pip install Jinja2
```

### Template HTML (`email_template.html`)

```html
<!DOCTYPE html>
<html>
<head>
    <style>
        body { font-family: Arial, sans-serif; }
        h1 { color: #2e6c80; }
    </style>
</head>
<body>
    <h1>Bonjour {{ nom }} !</h1>
    <p>Vous avez {{ nb_notifications }} nouvelles notifications.</p>
    <ul>
    {% for notif in notifications %}
        <li>{{ notif }}</li>
    {% endfor %}
    </ul>
    <p>Cordialement,<br>L'équipe</p>
</body>
</html>
```

### Code Python

```python
from jinja2 import Template
import smtplib
from email.message import EmailMessage

# Charger le template
with open('email_template.html', 'r', encoding='utf-8') as f:
    template = Template(f.read())

# Rendre le template avec des données
html_content = template.render(
    nom='Alice',
    nb_notifications=3,
    notifications=['Nouveau message', 'Commentaire sur votre post', 'Mise à jour système']
)

# Créer et envoyer l'email
msg = EmailMessage()
msg['Subject'] = 'Vos notifications'
msg['From'] = 'votre.email@gmail.com'
msg['To'] = 'alice@example.com'
msg.add_alternative(html_content, subtype='html')

with smtplib.SMTP('smtp.gmail.com', 587) as smtp:
    smtp.starttls()
    smtp.login('votre.email@gmail.com', 'votre_mot_de_passe')
    smtp.send_message(msg)
    print("Email avec template envoyé !")
```

---

## 13) Bonnes pratiques

### ✅ À faire

1. **Utiliser des mots de passe d'application** (Gmail, Yahoo) plutôt que le mot de passe principal
2. **Stocker les credentials dans des variables d'environnement** (`.env`)
3. **Gérer les erreurs** avec des try/except appropriés
4. **Ajouter un timeout** : `SMTP(..., timeout=10)`
5. **Utiliser STARTTLS ou SSL** pour chiffrer les connexions
6. **Valider les adresses email** avant l'envoi (regex ou lib `email-validator`)
7. **Limiter le taux d'envoi** pour éviter d'être banni (rate limiting)
8. **Respecter le RGPD** : permettre le désabonnement

### ❌ À éviter

- Hardcoder les mots de passe dans le code
- Envoyer des emails en masse sans throttling
- Ne pas gérer les exceptions
- Envoyer des emails non sécurisés (sans TLS/SSL)
- Oublier de fermer la connexion SMTP (utilisez `with`)

---

## 14) Cas d'usage pratiques

### 1. Notification d'erreur

```python
def notifier_erreur(exception):
    msg = EmailMessage()
    msg['Subject'] = f'[ERREUR] Application crash : {type(exception).__name__}'
    msg['From'] = 'monitoring@monapp.com'
    msg['To'] = 'admin@monapp.com'
    msg.set_content(f"Une erreur s'est produite :\n\n{str(exception)}")

    with smtplib.SMTP('smtp.gmail.com', 587) as smtp:
        smtp.starttls()
        smtp.login('monitoring@monapp.com', 'password')
        smtp.send_message(msg)
```

### 2. Rapport quotidien automatisé

```python
import schedule
import time

def envoyer_rapport_quotidien():
    # Générer le rapport
    rapport = generer_rapport()

    msg = EmailMessage()
    msg['Subject'] = f'Rapport quotidien - {date.today()}'
    msg['From'] = 'rapport@monapp.com'
    msg['To'] = 'direction@monapp.com'
    msg.set_content(rapport)

    with smtplib.SMTP('smtp.gmail.com', 587) as smtp:
        smtp.starttls()
        smtp.login('rapport@monapp.com', 'password')
        smtp.send_message(msg)
    print("Rapport envoyé")

# Programmer l'envoi tous les jours à 8h
schedule.every().day.at("08:00").do(envoyer_rapport_quotidien)

while True:
    schedule.run_pending()
    time.sleep(60)
```

### 3. Email de confirmation d'inscription

```python
def envoyer_confirmation_inscription(email_utilisateur, token):
    lien_confirmation = f"https://monapp.com/confirm?token={token}"

    html = f"""
    <html>
      <body>
        <h1>Bienvenue !</h1>
        <p>Merci pour votre inscription.</p>
        <p><a href="{lien_confirmation}">Confirmez votre email</a></p>
      </body>
    </html>
    """

    msg = EmailMessage()
    msg['Subject'] = 'Confirmez votre inscription'
    msg['From'] = 'noreply@monapp.com'
    msg['To'] = email_utilisateur
    msg.add_alternative(html, subtype='html')

    with smtplib.SMTP('smtp.gmail.com', 587) as smtp:
        smtp.starttls()
        smtp.login('noreply@monapp.com', 'password')
        smtp.send_message(msg)
```

---

## Conclusion

Python offre des outils puissants et flexibles pour envoyer des emails. Que vous utilisiez les modules natifs `smtplib` et `email` ou des bibliothèques tierces comme `yagmail` ou `Flask-Mail`, vous pouvez automatiser vos notifications facilement.

**Points clés à retenir :**

- `smtplib` + `email` : modules natifs, pas d'installation
- Configuration SMTP : Gmail (587/465), Outlook, serveurs personnalisés
- HTML + pièces jointes : `add_alternative()` + `add_attachment()`
- Sécurité : variables d'environnement, mots de passe d'application
- Gestion d'erreurs avec try/except
- Flask-Mail : intégration web
- Templates dynamiques avec Jinja2

Envoyez vos premiers emails automatisés dès maintenant ! 📧

---

## Voir aussi

- [Comment faire une API web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %})
- [Comment faire des requêtes HTTP en Python avec requests]({% post_url 2020-05-22-Comment-faire-des-requetes-http-en-python-avec-requests %})
- [Comment créer une CLI en Python]({% post_url 2025-12-28-Comment-creer-une-CLI-en-python %})
- [Comment faire une API web avec Flask]({% post_url 2021-04-20-Comment-faire-une-api-web-en-python %})
- [Documentation smtplib](https://docs.python.org/3/library/smtplib.html)
- [Documentation email](https://docs.python.org/3/library/email.html)
