# Offre Data — Pipeline de collecte d'offres d'emploi

![Python](https://img.shields.io/badge/Python-3.11-3776AB?logo=python&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Supabase-336791?logo=postgresql&logoColor=white)
![Statut](https://img.shields.io/badge/Statut-En%20d%C3%A9veloppement-orange)

## Présentation

**Offre Data** est un projet personnel de Data Engineering visant à collecter et centraliser des offres d'emploi liées à la **Data** en France, afin de préparer leur analyse.

À ce stade, le projet extrait des annonces de **Welcome to the Jungle** via un point d'accès de recherche **Algolia**, sélectionne les informations utiles avec **Pandas**, puis les charge dans une base **PostgreSQL hébergée sur Supabase**.

> **État du projet :** l'extraction et le stockage sont développés. L'enrichissement des données, leur analyse et leur visualisation restent à réaliser. Il s'agit actuellement d'un pipeline d'extraction et de chargement (**EL**), et non d'une chaîne ETL analytique complète.

## Fonctionnalités actuelles

- Recherche d'offres correspondant au mot-clé `data`, avec un filtre sur la France.
- Extraction des réponses JSON via une requête HTTP en Python.
- Sélection de quatre informations : **titre du poste, entreprise, département et ville**.
- Structuration des résultats dans un `DataFrame` Pandas.
- Chargement des données dans la table PostgreSQL `offres_data` via SQLAlchemy.
- Workflow **GitHub Actions** prévu pour une exécution quotidienne, avec déclenchement manuel possible.
- **Dockerfile** pour exécuter le script dans un conteneur.

## Architecture

```mermaid
flowchart TD
    A[Welcome to the Jungle / Algolia] --> B[Requête HTTP avec Requests]
    B --> C[Réponse JSON]
    C --> D[Extraction de 4 champs avec Pandas]
    D --> E[SQLAlchemy]
    E --> F[(PostgreSQL / Supabase)]
    G[GitHub Actions] -. lance .-> B
```

**Flux de traitement :** source d'offres → extraction Python → structuration Pandas → insertion en base PostgreSQL.

## Stack technique

| Technologie | Utilisation |
|---|---|
| **Python 3.11** | Exécution du pipeline |
| **Requests** | Requête HTTP vers la source de données |
| **Pandas** | Sélection et structuration des résultats |
| **SQLAlchemy** | Connexion à PostgreSQL et insertion des données |
| **PostgreSQL / Supabase** | Stockage des offres |
| **python-dotenv** | Lecture de la configuration locale |
| **GitHub Actions** | Planification et lancement automatisé |
| **Docker** | Exécution conteneurisée du script |

## Organisation du dépôt

```text
offre-data/
├── .github/
│   └── workflows/
│       └── pipeline.yml   # Automatisation GitHub Actions
├── main.py                # Extraction, structuration et chargement
├── requirements.txt       # Dépendances Python
├── Dockerfile             # Image d'exécution Python 3.11
├── .gitignore             # Fichiers à exclure de Git
└── README.md              # Documentation du projet
```

*Cette arborescence présente les fichiers utiles au fonctionnement du projet ; le dépôt contient également des fichiers de développement locaux.*

## Installation et exécution locale

### 1. Prérequis

- Python **3.11** (version utilisée dans le workflow et le Dockerfile).
- Une base de données **PostgreSQL**, par exemple sur **Supabase**.
- Git.

### 2. Cloner le dépôt

```bash
git clone https://github.com/FousseyniC/offre-data.git
cd offre-data
```

### 3. Créer un environnement virtuel et installer les dépendances

```bash
python -m venv .venv
```

Sur **Windows (PowerShell)** :

```powershell
.venv\Scripts\Activate.ps1
```

Sur **macOS / Linux** :

```bash
source .venv/bin/activate
```

Installer ensuite les bibliothèques :

```bash
pip install -r requirements.txt
```

### 4. Configurer PostgreSQL / Supabase

Créer à la racine du projet un fichier **`.env`** contenant une chaîne de connexion SQLAlchemy compatible avec PostgreSQL :

```dotenv
DATABASE_URL=postgresql+psycopg2://UTILISATEUR:MOT_DE_PASSE@HOTE:5432/NOM_BASE
```

Remplacer les valeurs d'exemple par les paramètres de connexion de votre base (port et hôte selon la configuration Supabase). Le script lit `DATABASE_URL` via `python-dotenv`.

**Sécurité :** ne jamais publier le fichier `.env`, vos mots de passe ou vos clés privées. Le fichier `.env` est ignoré par le `.gitignore` du projet.

### 5. Lancer le pipeline

```bash
python main.py
```

Si la requête aboutit et que la connexion PostgreSQL fonctionne, le script insère les résultats dans la table **`offres_data`**. Si elle n'existe pas encore, `pandas.to_sql()` peut la créer automatiquement.

## Données enregistrées

Le script stocke actuellement les champs suivants :

| Colonne | Description |
|---|---|
| `Titre du poste` | Intitulé de l'offre |
| `Entreprise` | Nom de l'entreprise |
| `Département` | Département ou information géographique renvoyée par la source |
| `Ville` | Ville indiquée dans l'offre |

**À noter :** dans l'implémentation actuelle, les offres sans bureau renseigné reçoivent la valeur `Télétravail` pour la ville et le département. Cette règle sera revue, car l'absence d'adresse ne prouve pas qu'un poste est en télétravail.

## Automatisation avec GitHub Actions

Le workflow [`.github/workflows/pipeline.yml`](.github/workflows/pipeline.yml) :

1. Récupère le code du dépôt.
2. Configure **Python 3.11**.
3. Installe les dépendances depuis `requirements.txt`.
4. Exécute `python main.py` avec la variable `DATABASE_URL`.

Le fichier YAML prévoit une exécution **tous les jours à 08:00 UTC** (`0 8 * * *`) et un déclenchement manuel via **Run workflow**.

Pour utiliser le workflow, ajouter dans les paramètres du dépôt GitHub un secret Actions nommé **`DATABASE_URL`** contenant la chaîne de connexion PostgreSQL. Si le workflow planifié a été désactivé par GitHub, il doit être réactivé depuis l'onglet **Actions**.

> La présence d'un planning dans le YAML ne garantit pas qu'une exécution a effectivement eu lieu ni que des données ont été insérées : consulter les journaux du workflow et la base cible.

## Exécution avec Docker

Le `Dockerfile` fourni utilise Python 3.11 et lance `main.py`.

```bash
docker build -t offre-data .
docker run --rm --env-file .env offre-data
```

Cette commande transmet la configuration de base de données au conteneur sans l'intégrer à l'image.

## Limites connues

Le projet est encore en développement :

- **Pagination absente** : seule la première page de résultats de la recherche principale est exploitée (jusqu'à 30 offres par exécution).
- **Doublons possibles** : les nouvelles extractions sont ajoutées avec `if_exists="append"`, sans contrôle d'unicité.
- **Gestion des erreurs à renforcer** : délais d'attente HTTP, exceptions et remontée explicite des échecs.
- **Données à enrichir** : identifiants d'offres, dates, contrats, liens et compétences ne sont pas encore stockés.
- **Analyse non développée** : pas encore de requêtes analytiques, de tableau de bord ou de visualisation.
- **Source susceptible d'évoluer** : le point d'accès Algolia utilisé par la recherche de Welcome to the Jungle peut changer ; son utilisation doit respecter les conditions applicables au site.

## Feuille de route

### Déjà réalisé

- [x] Récupérer les offres d'emploi avec Python.
- [x] Enregistrer les données dans Supabase (PostgreSQL).
- [x] Configurer l'automatisation avec GitHub Actions.

### Prochaines étapes

- [ ] Fiabiliser la collecte (erreurs et doublons).
- [ ] Nettoyer et compléter les données.
- [ ] Analyser les offres avec SQL.
- [ ] Créer un tableau de bord pour visualiser les résultats.

## Auteur

**Fousseyni Cissé**  
Étudiant en informatique, spécialisé en Data et Intelligence Artificielle.  
GitHub : [@FousseyniC](https://github.com/FousseyniC)

---

*Projet personnel pédagogique, indépendant de Welcome to the Jungle et de Supabase.*
