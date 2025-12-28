---
layout: article
title: "Python : Comment créer une CLI"
author: Pierre Chopinet
tags:
  - python
  - cli
  - argparse
  - click
  - typer
---

Dans ce tutoriel, vous allez apprendre à créer une interface en ligne de commande (CLI) en Python. <!--more--> Nous verrons trois approches différentes :

- `argparse` : la bibliothèque standard de Python
- `click` : une bibliothèque populaire et intuitive
- `typer` : une bibliothèque moderne basée sur les type hints

## Pourquoi créer une CLI ?

Une CLI permet de rendre vos scripts Python interactifs et configurables sans modifier le code. Les avantages :

- Passer des paramètres facilement
- Créer des outils réutilisables
- Automatiser des tâches dans des scripts bash ou pipelines CI/CD
- Fournir une interface utilisateur simple et efficace

## Méthode 1 : argparse (bibliothèque standard)

`argparse` est inclus dans Python, aucune installation nécessaire.

### Exemple basique

```python
import argparse

parser = argparse.ArgumentParser(description="Un outil CLI simple")
parser.add_argument("nom", help="Votre nom")
parser.add_argument("-a", "--age", type=int, help="Votre âge")

args = parser.parse_args()

print(f"Bonjour {args.nom}!")
if args.age:
    print(f"Vous avez {args.age} ans.")
```

Utilisation :

```bash
python script.py Pierre --age 25
# Bonjour Pierre!
# Vous avez 25 ans.
```

### Arguments positionnels vs optionnels

```python
import argparse

parser = argparse.ArgumentParser(description="Gestionnaire de fichiers")

# Argument positionnel (obligatoire)
parser.add_argument("fichier", help="Chemin du fichier")

# Arguments optionnels
parser.add_argument("-o", "--output", help="Fichier de sortie")
parser.add_argument("-v", "--verbose", action="store_true", help="Mode verbeux")
parser.add_argument("-n", "--nombre", type=int, default=10, help="Nombre de lignes (défaut: 10)")

args = parser.parse_args()

print(f"Fichier: {args.fichier}")
print(f"Output: {args.output}")
print(f"Verbose: {args.verbose}")
print(f"Nombre: {args.nombre}")
```

### Sous-commandes avec argparse

```python
import argparse

parser = argparse.ArgumentParser(description="Gestionnaire de tâches")
subparsers = parser.add_subparsers(dest="commande", help="Commandes disponibles")

# Sous-commande "add"
parser_add = subparsers.add_parser("add", help="Ajouter une tâche")
parser_add.add_argument("tache", help="Description de la tâche")
parser_add.add_argument("-p", "--priorite", type=int, default=1, help="Priorité (1-5)")

# Sous-commande "list"
parser_list = subparsers.add_parser("list", help="Lister les tâches")
parser_list.add_argument("-a", "--all", action="store_true", help="Afficher toutes les tâches")

# Sous-commande "delete"
parser_delete = subparsers.add_parser("delete", help="Supprimer une tâche")
parser_delete.add_argument("id", type=int, help="ID de la tâche")

args = parser.parse_args()

if args.commande == "add":
    print(f"Ajout de la tâche: {args.tache} (priorité: {args.priorite})")
elif args.commande == "list":
    print(f"Liste des tâches (all={args.all})")
elif args.commande == "delete":
    print(f"Suppression de la tâche #{args.id}")
else:
    parser.print_help()
```

Utilisation :

```bash
python tasks.py add "Finir le rapport" -p 3
python tasks.py list --all
python tasks.py delete 5
```

## Méthode 2 : Click

`click` est une bibliothèque qui simplifie la création de CLI avec des décorateurs.

### Installation

```bash
pip install click
```

### Exemple basique

```python
import click

@click.command()
@click.argument("nom")
@click.option("-a", "--age", type=int, help="Votre âge")
def bonjour(nom, age):
    """Un outil CLI simple."""
    click.echo(f"Bonjour {nom}!")
    if age:
        click.echo(f"Vous avez {age} ans.")

if __name__ == "__main__":
    bonjour()
```

### Options avancées avec Click

```python
import click

@click.command()
@click.argument("fichier", type=click.Path(exists=True))
@click.option("-o", "--output", type=click.Path(), help="Fichier de sortie")
@click.option("-v", "--verbose", is_flag=True, help="Mode verbeux")
@click.option("-f", "--format", type=click.Choice(["json", "csv", "xml"]), default="json")
@click.option("-n", "--nombre", type=int, default=10, show_default=True)
def traiter(fichier, output, verbose, format, nombre):
    """Traite un fichier et génère une sortie."""
    if verbose:
        click.echo(f"Traitement de {fichier}...")

    click.echo(f"Format: {format}, Lignes: {nombre}")

    if output:
        click.echo(f"Écriture vers {output}")

if __name__ == "__main__":
    traiter()
```

### Groupes de commandes avec Click

```python
import click

@click.group()
def cli():
    """Gestionnaire de tâches."""
    pass

@cli.command()
@click.argument("tache")
@click.option("-p", "--priorite", type=int, default=1, help="Priorité (1-5)")
def add(tache, priorite):
    """Ajouter une nouvelle tâche."""
    click.echo(f"Ajout: {tache} (priorité: {priorite})")

@cli.command()
@click.option("-a", "--all", "show_all", is_flag=True, help="Afficher toutes les tâches")
def list(show_all):
    """Lister les tâches."""
    click.echo(f"Liste des tâches (all={show_all})")

@cli.command()
@click.argument("task_id", type=int)
@click.confirmation_option(prompt="Êtes-vous sûr de vouloir supprimer?")
def delete(task_id):
    """Supprimer une tâche."""
    click.echo(f"Suppression de la tâche #{task_id}")

if __name__ == "__main__":
    cli()
```

### Couleurs et style avec Click

```python
import click

@click.command()
def status():
    """Affiche le statut avec des couleurs."""
    click.secho("Succès!", fg="green", bold=True)
    click.secho("Attention!", fg="yellow")
    click.secho("Erreur!", fg="red", bold=True)

    # Barre de progression
    with click.progressbar(range(100), label="Traitement") as items:
        for item in items:
            # Simuler un traitement
            import time
            time.sleep(0.01)

if __name__ == "__main__":
    status()
```

## Méthode 3 : Typer (moderne et typé)

`typer` utilise les type hints de Python pour générer automatiquement une CLI.

### Installation

```bash
pip install typer
```

### Exemple basique

```python
import typer

def main(nom: str, age: int = None):
    """Un outil CLI simple."""
    print(f"Bonjour {nom}!")
    if age:
        print(f"Vous avez {age} ans.")

if __name__ == "__main__":
    typer.run(main)
```

C'est tout! Typer déduit automatiquement les types et génère l'aide.

### Options avancées avec Typer

```python
import typer
from typing import Optional
from pathlib import Path
from enum import Enum

class Format(str, Enum):
    json = "json"
    csv = "csv"
    xml = "xml"

def traiter(
    fichier: Path = typer.Argument(..., help="Chemin du fichier"),
    output: Optional[Path] = typer.Option(None, "--output", "-o", help="Fichier de sortie"),
    verbose: bool = typer.Option(False, "--verbose", "-v", help="Mode verbeux"),
    format: Format = typer.Option(Format.json, "--format", "-f", help="Format de sortie"),
    nombre: int = typer.Option(10, "--nombre", "-n", help="Nombre de lignes"),
):
    """Traite un fichier et génère une sortie."""
    if verbose:
        typer.echo(f"Traitement de {fichier}...")

    typer.echo(f"Format: {format.value}, Lignes: {nombre}")

    if output:
        typer.echo(f"Écriture vers {output}")

if __name__ == "__main__":
    typer.run(traiter)
```

### Application avec sous-commandes

```python
import typer
from typing import Optional

app = typer.Typer(help="Gestionnaire de tâches")

@app.command()
def add(
    tache: str = typer.Argument(..., help="Description de la tâche"),
    priorite: int = typer.Option(1, "--priorite", "-p", min=1, max=5, help="Priorité"),
):
    """Ajouter une nouvelle tâche."""
    typer.echo(f"Ajout: {tache} (priorité: {priorite})")

@app.command()
def list(
    all: bool = typer.Option(False, "--all", "-a", help="Afficher toutes les tâches"),
):
    """Lister les tâches."""
    typer.echo(f"Liste des tâches (all={all})")

@app.command()
def delete(
    task_id: int = typer.Argument(..., help="ID de la tâche"),
    force: bool = typer.Option(False, "--force", "-f", help="Supprimer sans confirmation"),
):
    """Supprimer une tâche."""
    if not force:
        confirm = typer.confirm("Êtes-vous sûr?")
        if not confirm:
            raise typer.Abort()
    typer.echo(f"Suppression de la tâche #{task_id}")

if __name__ == "__main__":
    app()
```

### Couleurs et style avec Typer

```python
import typer

def main():
    typer.secho("Succès!", fg=typer.colors.GREEN, bold=True)
    typer.secho("Attention!", fg=typer.colors.YELLOW)
    typer.secho("Erreur!", fg=typer.colors.RED, bold=True)

    # Demander une entrée
    nom = typer.prompt("Quel est votre nom?")
    typer.echo(f"Bonjour {nom}!")

    # Confirmation
    if typer.confirm("Voulez-vous continuer?"):
        typer.echo("Continuation...")

if __name__ == "__main__":
    typer.run(main)
```

## Rendre votre CLI installable

Pour distribuer votre CLI, créez un fichier `pyproject.toml` :

```toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[project]
name = "mon-outil"
version = "1.0.0"
description = "Mon outil CLI"
dependencies = [
    "typer>=0.9.0",
]

[project.scripts]
mon-outil = "mon_package.cli:app"
```

Installez ensuite avec :

```bash
pip install -e .
```

Votre commande `mon-outil` sera disponible globalement.

## Comparaison des bibliothèques

| Fonctionnalité         | argparse   | click       | typer       |
|------------------------|------------|-------------|-------------|
| Installation           | Inclus     | pip         | pip         |
| Syntaxe                | Impérative | Décorateurs | Type hints  |
| Courbe d'apprentissage | Moyenne    | Facile      | Très facile |
| Sous-commandes         | Oui        | Oui         | Oui         |
| Couleurs/style         | Non        | Oui         | Oui         |
| Autocomplétion         | Non        | Plugin      | Intégré     |
| Validation             | Basique    | Avancée     | Avancée     |

**Recommandations :**

- **argparse** : Pour des scripts simples sans dépendances externes
- **click** : Pour des CLI complexes avec beaucoup de personnalisation
- **typer** : Pour des CLI modernes avec une syntaxe propre et typée

## Voir aussi

- Documentation argparse : [https://docs.python.org/3/library/argparse.html](https://docs.python.org/3/library/argparse.html)
- Documentation click : [https://click.palletsprojects.com](https://click.palletsprojects.com)
- Documentation typer : [https://typer.tiangolo.com](https://typer.tiangolo.com)
- [Python : Comment faire des requêtes HTTP avec requests]({% post_url 2020-05-22-Comment-faire-des-requetes-http-en-python-avec-requests %})
- [Python : Comment faire une api web avec FastAPI]({% post_url 2025-08-15-Comment-faire-une-api-web-avec-FastAPI %})
