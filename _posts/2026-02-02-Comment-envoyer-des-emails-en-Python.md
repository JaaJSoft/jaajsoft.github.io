---
layout: article
title: "Python : Comment envoyer des emails"
tags:
  - python
  - email
  - smtp
author: Pierre Chopinet
---

Python sait envoyer des emails sans rien installer, avec les modules `smtplib` et `email` de la bibliothèque standard. Dans ce tutoriel, nous allons envoyer un email texte puis HTML, ajouter des pièces jointes, gérer les erreurs, et voir comment faire la même chose depuis une application Flask.
<!--more-->

Dans cet article :
- Envoyer un email simple
- Les paramètres SMTP des principaux fournisseurs
- Envoyer un email HTML
- Envoyer à plusieurs destinataires
- Ajouter des pièces jointes
- Gérer les erreurs
- Utiliser SSL sur le port 465
- Ne pas écrire le mot de passe dans le code
- Envoyer des emails depuis Flask avec Flask-Mail
- Envoyer un email sans bloquer le programme
- Des templates d'emails avec Jinja2
- Exemples d'emails automatiques

Pré-requis : Python 3, `smtplib` et `email` sont inclus dans la bibliothèque standard. Les exemples ont été testés avec Python 3.13, et avec Flask-Mail 0.10.0, python-dotenv 1.2.4 et Jinja2 3.1.6 pour les parties qui les utilisent.

## Envoyer un email simple

On construit le message avec `EmailMessage`, puis on l'envoie avec `smtplib` :

```python
import smtplib
import ssl
from email.message import EmailMessage

# Créer le message
msg = EmailMessage()
msg['Subject'] = 'Test depuis Python'
msg['From'] = 'votre.email@gmail.com'
msg['To'] = 'destinataire@example.com'
msg.set_content('Ceci est un email de test envoyé depuis Python.')

# Envoyer via SMTP
context = ssl.create_default_context()
with smtplib.SMTP('smtp.gmail.com', 587) as smtp:
    smtp.starttls(context=context)  # Connexion chiffrée
    smtp.login('votre.email@gmail.com', 'votre_mot_de_passe')
    smtp.send_message(msg)
    print("Email envoyé avec succès !")
```

Les en-têtes (`Subject`, `From`, `To`) s'affectent comme les clés d'un dictionnaire, et `set_content()` définit le corps du message en texte brut. Pour l'envoi, on se connecte au serveur SMTP de Gmail sur le port 587 : la connexion démarre en clair, `starttls()` la fait passer en TLS, puis `login()` s'authentifie et `send_message()` envoie le message. À la sortie du bloc `with`, la connexion est fermée proprement.

Attention au paramètre `context` de `starttls()` : sans lui, la connexion est bien chiffrée mais le certificat du serveur n'est pas vérifié, ce qui laisse la porte ouverte à une attaque de type *man-in-the-middle*. `ssl.create_default_context()` active cette vérification, c'est d'ailleurs ce que recommande la documentation de Python.

### Tester sans envoyer de vrais emails

Pour faire des essais sans spammer personne, on peut lancer un faux serveur SMTP en local avec aiosmtpd (ici en version 1.4.6), qui affiche dans le terminal les messages qu'il reçoit. L'option `-n` est nécessaire si vous ne lancez pas la commande en root :

```bash
pip install aiosmtpd
python -m aiosmtpd -n -l 127.0.0.1:8025
```

Ce serveur ne gère ni le chiffrement ni l'authentification, on retire donc `starttls()` et `login()` :

```python
with smtplib.SMTP('127.0.0.1', 8025) as smtp:
    smtp.send_message(msg)
```

Le serveur affiche alors le message tel qu'il l'a reçu (le port indiqué par `X-Peer` change à chaque connexion) :

```
---------- MESSAGE FOLLOWS ----------
Subject: Test depuis Python
From: votre.email@gmail.com
To: destinataire@example.com
Content-Type: text/plain; charset="utf-8"
Content-Transfer-Encoding: 8bit
MIME-Version: 1.0
X-Peer: ('127.0.0.1', 54260)

Ceci est un email de test envoyé depuis Python.
------------ END MESSAGE ------------
```

C'est aussi un bon moyen de voir à quoi ressemble un message HTML ou avec pièces jointes une fois construit.

## Les paramètres SMTP des principaux fournisseurs

`smtplib` fonctionne avec n'importe quel serveur SMTP, il suffit de changer l'adresse et le port :

| Fournisseur       | Serveur SMTP            | Port (STARTTLS) |
|-------------------|-------------------------|-----------------|
| Gmail             | `smtp.gmail.com`        | 587             |
| Outlook / Hotmail | `smtp-mail.outlook.com` | 587             |
| Office 365        | `smtp.office365.com`    | 587             |
| Yahoo Mail        | `smtp.mail.yahoo.com`   | 587             |

Pour votre propre domaine, utilisez le serveur indiqué par votre hébergeur (`mail.mondomaine.com` par exemple), sur le port 587, ou 465 pour une connexion SSL directe (voir plus bas).

Pour Gmail, il faut un mot de passe d'application : activez la validation en deux étapes dans [Compte Google > Sécurité](https://myaccount.google.com/security), puis générez un "mot de passe d'application" que vous utiliserez à la place de votre mot de passe habituel. Yahoo fonctionne aussi avec un mot de passe d'application. Chez Microsoft, c'est plus compliqué : depuis le 16 septembre 2024, les comptes Outlook.com et Hotmail n'acceptent plus de connexion par mot de passe pour les applications tierces, il faut passer par OAuth2, ce que `smtplib.login()` ne sait pas faire seul. Pour Office 365, Microsoft a annoncé la désactivation par défaut de l'authentification SMTP par mot de passe à partir de fin 2026. Plus généralement, les méthodes d'authentification acceptées varient d'un fournisseur à l'autre et évoluent : si la connexion est refusée, vérifiez la documentation de votre fournisseur.

## Envoyer un email HTML

```python
import smtplib
import ssl
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
    smtp.starttls(context=ssl.create_default_context())
    smtp.login('votre.email@gmail.com', 'votre_mot_de_passe')
    smtp.send_message(msg)
    print("Email HTML envoyé !")
```

`set_content()` définit la version texte, et `add_alternative(..., subtype='html')` ajoute la version HTML. Le message contient alors les deux versions : le client mail affiche le HTML s'il en est capable, sinon le texte brut.

## Envoyer à plusieurs destinataires

```python
msg = EmailMessage()
msg['Subject'] = 'Réunion d\'équipe'
msg['From'] = 'votre.email@gmail.com'
msg['To'] = 'alice@example.com, bob@example.com'  # Liste séparée par virgules
msg['Cc'] = 'manager@example.com'  # Copie
msg['Bcc'] = 'archive@example.com'  # Copie cachée

msg.set_content('Rappel : réunion demain à 10h.')

with smtplib.SMTP('smtp.gmail.com', 587) as smtp:
    smtp.starttls(context=ssl.create_default_context())
    smtp.login('votre.email@gmail.com', 'votre_mot_de_passe')
    smtp.send_message(msg)
```

`send_message()` envoie le message à toutes les adresses de `To`, `Cc` et `Bcc`, mais ne transmet pas l'en-tête `Bcc` lui-même : les autres destinataires ne voient pas les copies cachées.

Si les adresses sont dans une liste Python, on les joint avec une virgule :

```python
destinataires = ['alice@example.com', 'bob@example.com', 'charlie@example.com']
msg['To'] = ', '.join(destinataires)
```

Pour un envoi à beaucoup de monde, espacez les envois : les fournisseurs limitent le nombre d'emails envoyés et peuvent bloquer un compte qui en envoie trop.

## Ajouter des pièces jointes

Une pièce jointe s'ajoute avec `add_attachment()`, en lui passant le contenu du fichier en bytes, son type MIME et son nom :

```python
import smtplib
import ssl
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
    smtp.starttls(context=ssl.create_default_context())
    smtp.login('votre.email@gmail.com', 'votre_mot_de_passe')
    smtp.send_message(msg)
    print(f"Email avec {file_name} envoyé !")
```

### Plusieurs pièces jointes

Pour joindre plusieurs fichiers de types différents, on peut laisser le module `mimetypes` deviner le type à partir de l'extension, avec `application/octet-stream` (fichier binaire quelconque) quand il ne le connaît pas :

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

### Image intégrée dans le HTML

Pour afficher une image dans le corps du message plutôt qu'en pièce jointe, on la rattache à la partie HTML avec un identifiant (`Content-ID`), auquel le HTML fait référence avec `cid:` :

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

`msg.get_payload()[0]` est la partie HTML, à laquelle `add_related()` rattache l'image. Si vous ajoutez une version texte avec `set_content()` avant `add_alternative()`, la partie HTML devient `msg.get_payload()[1]`.

## Gérer les erreurs

```python
import smtplib
import ssl
from email.message import EmailMessage

try:
    msg = EmailMessage()
    msg['Subject'] = 'Test'
    msg['From'] = 'votre.email@gmail.com'
    msg['To'] = 'destinataire@example.com'
    msg.set_content('Test')

    with smtplib.SMTP('smtp.gmail.com', 587, timeout=10) as smtp:
        smtp.starttls(context=ssl.create_default_context())
        smtp.login('votre.email@gmail.com', 'votre_mot_de_passe')
        smtp.send_message(msg)
        print("Email envoyé avec succès")

except smtplib.SMTPAuthenticationError:
    print("Erreur d'authentification (email/mot de passe incorrect)")
except smtplib.SMTPException as e:
    print(f"Erreur SMTP : {e}")
except Exception as e:
    print(f"Erreur : {e}")
```

Le paramètre `timeout` évite de rester bloqué indéfiniment si le serveur ne répond pas. Les exceptions que vous rencontrerez le plus souvent :

- `SMTPAuthenticationError` : identifiants incorrects
- `SMTPRecipientsRefused` : le serveur a refusé tous les destinataires
- `SMTPServerDisconnected` : connexion perdue
- `socket.gaierror` : serveur SMTP introuvable

Toutes les exceptions de `smtplib` héritent de `SMTPException`, elle-même sous-classe d'`OSError`. Les erreurs réseau (serveur introuvable, connexion refusée, timeout) ne sont pas des `SMTPException` : ce sont des `OSError`, attrapées ici par le dernier `except`.

Si le serveur n'accepte qu'une partie des destinataires, il n'y a pas d'exception : `send_message()` renvoie un dictionnaire avec les adresses refusées et l'erreur associée. Un dictionnaire vide signifie que tout le monde a été accepté.

## Utiliser SSL sur le port 465

Tous les exemples précédents utilisent `starttls()` sur le port 587. Certains serveurs acceptent aussi une connexion chiffrée dès le départ sur le port 465, avec `SMTP_SSL` :

```python
import smtplib
import ssl
from email.message import EmailMessage

msg = EmailMessage()
msg['Subject'] = 'Test SSL'
msg['From'] = 'votre.email@gmail.com'
msg['To'] = 'destinataire@example.com'
msg.set_content('Test avec SSL')

# SMTP_SSL sur le port 465
with smtplib.SMTP_SSL('smtp.gmail.com', 465, context=ssl.create_default_context()) as smtp:
    smtp.login('votre.email@gmail.com', 'votre_mot_de_passe')
    smtp.send_message(msg)
    print("Email envoyé via SSL")
```

Sur le port 587, la connexion démarre en clair puis passe en TLS avec STARTTLS. Sur le port 465, elle est chiffrée dès le début (on parle de TLS implicite). Longtemps considéré comme obsolète, ce second mode est de nouveau recommandé par la RFC 8314 (2018). Comme pour `starttls()`, passez un contexte SSL : sans lui, `SMTP_SSL` ne vérifie pas non plus le certificat du serveur.

## Ne pas écrire le mot de passe dans le code

Les exemples précédents contiennent le mot de passe en clair pour rester lisibles. Dans un vrai projet, on le range dans une variable d'environnement, par exemple avec python-dotenv :

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

Fichier `.env` :

```
EMAIL=votre.email@gmail.com
EMAIL_PASSWORD=abcd efgh ijkl mnop
```

Installation :

```bash
pip install python-dotenv
```

Pensez à ajouter `.env` à votre `.gitignore` pour ne pas committer vos secrets. L'article [Python : Comment utiliser les variables d'environnement avec python-dotenv]({% post_url 2026-04-20-Comment-utiliser-les-variables-d-environnement-avec-python-dotenv %}) détaille le fonctionnement.

## Envoyer des emails depuis Flask avec Flask-Mail

Dans une application Flask, on peut passer par l'extension Flask-Mail :

```bash
pip install Flask-Mail
```

Elle lit les paramètres SMTP dans la configuration de l'application :

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

`MAIL_DEFAULT_SENDER` sert d'expéditeur quand le `Message` n'en précise pas, et `body` et `html` jouent le même rôle que `set_content()` et `add_alternative()`. Attention, Flask-Mail 0.10.0 appelle `starttls()` sans contexte SSL : le certificat du serveur n'est donc pas vérifié. Si c'est un problème dans votre cas, passez par `smtplib` comme dans le reste de l'article.

## Envoyer un email sans bloquer le programme

L'envoi d'un email demande plusieurs échanges avec le serveur SMTP, ce qui peut prendre un moment. Pour ne pas bloquer le reste du programme pendant ce temps, on peut l'envoyer dans un thread :

```python
import smtplib
import ssl
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
            smtp.starttls(context=ssl.create_default_context())
            smtp.login('votre.email@gmail.com', 'votre_mot_de_passe')
            smtp.send_message(msg)
            print(f"Email envoyé à {destinataire}")

    thread = threading.Thread(target=_send)
    thread.start()

# Utilisation
envoyer_email_async('destinataire@example.com', 'Test async', 'Message de test')
print("L'envoi est en cours en arrière-plan...")
```

Le message "L'envoi est en cours en arrière-plan..." s'affiche avant la confirmation d'envoi. Par contre, une exception levée dans le thread n'arrive pas jusqu'au programme principal : elle est seulement affichée sur la sortie d'erreur.

Dans un programme `asyncio`, on exécute la fonction d'envoi, qui est bloquante, dans un thread avec `run_in_executor()` :

```python
import asyncio
import smtplib
import ssl
from email.message import EmailMessage

async def envoyer_email_async(destinataire, sujet, contenu):
    loop = asyncio.get_running_loop()
    await loop.run_in_executor(None, envoyer_email_sync, destinataire, sujet, contenu)

def envoyer_email_sync(destinataire, sujet, contenu):
    msg = EmailMessage()
    msg['Subject'] = sujet
    msg['From'] = 'votre.email@gmail.com'
    msg['To'] = destinataire
    msg.set_content(contenu)

    with smtplib.SMTP('smtp.gmail.com', 587) as smtp:
        smtp.starttls(context=ssl.create_default_context())
        smtp.login('votre.email@gmail.com', 'votre_mot_de_passe')
        smtp.send_message(msg)

# Utilisation
asyncio.run(envoyer_email_async('destinataire@example.com', 'Test', 'Message'))
```

## Des templates d'emails avec Jinja2

Plutôt que de construire le HTML à la main dans le code, on peut l'écrire dans un template Jinja2 et n'injecter que les données.

```bash
pip install Jinja2
```

Le template, dans un fichier `email_template.html` :

{% raw %}
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
{% endraw %}

Le code Python qui le remplit et envoie l'email :

```python
from jinja2 import Template
import smtplib
import ssl
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
    smtp.starttls(context=ssl.create_default_context())
    smtp.login('votre.email@gmail.com', 'votre_mot_de_passe')
    smtp.send_message(msg)
    print("Email avec template envoyé !")
```

Attention, `Template` n'échappe pas le HTML par défaut. Si les données viennent de vos utilisateurs (un nom saisi dans un formulaire par exemple), créez le template avec `Template(f.read(), autoescape=True)` pour qu'un `<script>` glissé dans un nom s'affiche comme du texte.

## Exemples d'emails automatiques

Pour finir, trois cas courants qui reprennent ce qu'on a vu plus haut.

### Être prévenu d'une erreur

```python
import smtplib
import ssl
from email.message import EmailMessage

def notifier_erreur(exception):
    msg = EmailMessage()
    msg['Subject'] = f'[ERREUR] Application crash : {type(exception).__name__}'
    msg['From'] = 'monitoring@monapp.com'
    msg['To'] = 'admin@monapp.com'
    msg.set_content(f"Une erreur s'est produite :\n\n{str(exception)}")

    with smtplib.SMTP('smtp.gmail.com', 587) as smtp:
        smtp.starttls(context=ssl.create_default_context())
        smtp.login('monitoring@monapp.com', 'password')
        smtp.send_message(msg)
```

### Un rapport quotidien

Cet exemple utilise la bibliothèque `schedule` (`pip install schedule`) pour lancer l'envoi tous les jours à 8 h. La fonction `generer_rapport()` est à écrire selon vos besoins.

```python
import schedule
import smtplib
import ssl
import time
from datetime import date
from email.message import EmailMessage

def envoyer_rapport_quotidien():
    # Générer le rapport
    rapport = generer_rapport()

    msg = EmailMessage()
    msg['Subject'] = f'Rapport quotidien - {date.today()}'
    msg['From'] = 'rapport@monapp.com'
    msg['To'] = 'direction@monapp.com'
    msg.set_content(rapport)

    with smtplib.SMTP('smtp.gmail.com', 587) as smtp:
        smtp.starttls(context=ssl.create_default_context())
        smtp.login('rapport@monapp.com', 'password')
        smtp.send_message(msg)
    print("Rapport envoyé")

# Programmer l'envoi tous les jours à 8h
schedule.every().day.at("08:00").do(envoyer_rapport_quotidien)

while True:
    schedule.run_pending()
    time.sleep(60)
```

Le script doit tourner en permanence pour que l'envoi ait lieu. Sur un serveur Linux, une tâche cron qui lance le script une fois par jour est souvent plus simple, voir [Linux : Programmer une tâche avec cron]({% post_url 2025-10-11-Linux-programmer-une-tache-avec-cron %}).

### Un email de confirmation d'inscription

```python
import smtplib
import ssl
from email.message import EmailMessage

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
        smtp.starttls(context=ssl.create_default_context())
        smtp.login('noreply@monapp.com', 'password')
        smtp.send_message(msg)
```

## Voir aussi

- [Python : Comment utiliser les variables d'environnement avec python-dotenv]({% post_url 2026-04-20-Comment-utiliser-les-variables-d-environnement-avec-python-dotenv %})
- [Python : Comment faire une api web avec Flask]({% post_url 2021-04-20-Comment-faire-une-api-web-en-python %})
- [Linux : Programmer une tâche avec cron]({% post_url 2025-10-11-Linux-programmer-une-tache-avec-cron %})
- [Documentation de smtplib](https://docs.python.org/3/library/smtplib.html)
- [Documentation du module email (avec des exemples)](https://docs.python.org/3/library/email.examples.html)
