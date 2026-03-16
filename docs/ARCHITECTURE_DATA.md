# Architecture Data & BDD

> Ce document decrit la structure du code pour les modules **ETL** et **Base de Donnees**.
> Il sert de reference pour le developpeur Data (Dev 1) et le setup initial du projet.

---

## Structure du repository

```
healthai-coach/
│
├── docker-compose.yml              # Orchestration de tous les services
├── .env.example                    # Variables d'environnement template
├── .gitignore
│
├── db/                             # --- MODULE BASE DE DONNEES ---
│   ├── migrations/                 # Scripts Flyway (executes par Spring Boot)
│   │   ├── V1__create_users.sql
│   │   ├── V2__create_nutrition_entries.sql
│   │   ├── V3__create_exercises.sql
│   │   ├── V4__create_exercise_entries.sql
│   │   ├── V5__create_biometric_entries.sql
│   │   └── V6__create_etl_logs.sql
│   └── seed/                       # Donnees initiales (optionnel, pour dev/demo)
│       └── seed_admin_user.sql
│
├── etl/                            # --- MODULE ETL PYTHON ---
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── .env.example
│   │
│   ├── main.py                     # Point d'entree : orchestre le pipeline
│   ├── config.py                   # Configuration (BDD, chemins, constantes)
│   │
│   ├── extractors/                 # Etape 1 : lecture des sources
│   │   ├── __init__.py
│   │   ├── csv_extractor.py        # Lecture CSV generique (Pandas)
│   │   └── json_extractor.py       # Lecture JSON imbrique (ExerciseDB)
│   │
│   ├── cleaners/                   # Etape 2-3 : validation + nettoyage
│   │   ├── __init__.py
│   │   ├── base_cleaner.py         # Classe abstraite : interface commune
│   │   ├── nutrition_cleaner.py    # Regles metier Daily Food & Nutrition
│   │   ├── diet_reco_cleaner.py    # Regles metier Diet Recommendations
│   │   ├── exercise_cleaner.py     # Regles metier ExerciseDB (aplatissement JSON)
│   │   ├── gym_members_cleaner.py  # Regles metier Gym Members
│   │   └── fitness_tracker_cleaner.py  # Regles metier Fitness Tracker + fusion
│   │
│   ├── loaders/                    # Etape 4 : chargement en base
│   │   ├── __init__.py
│   │   └── postgres_loader.py      # Insert/upsert PostgreSQL
│   │
│   ├── generators/                 # Donnees simulees
│   │   ├── __init__.py
│   │   └── fake_data_generator.py  # Faker : users + metriques business
│   │
│   ├── utils/                      # Utilitaires partages
│   │   ├── __init__.py
│   │   └── logger.py               # Logging fichier + BDD (etl_logs)
│   │
│   └── tests/                      # Tests unitaires des cleaners
│       ├── __init__.py
│       ├── test_nutrition_cleaner.py
│       ├── test_exercise_cleaner.py
│       └── test_gym_members_cleaner.py
│
├── data/                           # --- DATASETS BRUTS (git-ignored sauf samples) ---
│   ├── raw/                        # Fichiers telecharges (NON versionnes)
│   │   ├── .gitkeep
│   │   ├── daily_food_nutrition.csv
│   │   ├── diet_recommendations.csv
│   │   ├── gym_members_exercise.csv
│   │   ├── fitness_tracker.csv
│   │   └── exercisedb/
│   │       └── exercises.json
│   ├── samples/                    # Echantillons versionnes (10-20 lignes pour les tests)
│   │   ├── daily_food_nutrition_sample.csv
│   │   ├── diet_recommendations_sample.csv
│   │   ├── gym_members_exercise_sample.csv
│   │   └── exercises_sample.json
│   └── processed/                  # Exports apres nettoyage (NON versionnes)
│       └── .gitkeep
│
├── api/                            # (Dev 2-3 : Spring Boot - hors scope Data)
│
├── frontend/                       # (Dev 4 : React - hors scope Data)
│
└── docs/                           # Documentation projet
    ├── PLAN_GLOBAL.md
    ├── data/
    │   └── SOURCES.md
    └── adr/
        └── ...
```

---

## Separation des responsabilites

```
Dev 1 (Data)           Dev 2-3 (Backend)         Dev 4 (Frontend)
─────────────          ──────────────────         ─────────────────
etl/                   api/                       frontend/
data/                  db/migrations/*
db/migrations/*          (V1-V6 ecrits
  (V1-V6 proposes         ensemble, appliques
   par Data,               par Flyway via
   valides avec            Spring Boot)
   Backend)
```

### Convention importante

Les **migrations SQL** (`db/migrations/`) sont ecrites en collaboration entre Dev 1 (Data) et Dev 2 (Backend) :
- Dev 1 propose le schema base sur l'analyse des datasets reels
- Dev 2 valide et ajuste pour l'ORM JPA (noms, types, contraintes)
- Flyway les execute automatiquement au demarrage de Spring Boot
- L'ETL Python doit respecter exactement ce schema pour ses insertions

---

## Docker Compose (perimetre Data + BDD)

Pour travailler en local sans attendre l'API Spring Boot, Dev 1 demarre uniquement :

```bash
# Demarrer uniquement la base de donnees
docker compose up db -d

# Lancer le pipeline ETL en local (hors Docker, pour le dev)
cd etl
pip install -r requirements.txt
python main.py

# OU demarrer le conteneur ETL complet
docker compose up etl
```

### Services Data dans le docker-compose.yml

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ${DB_NAME:-healthai}
      POSTGRES_USER: ${DB_USER:-healthai}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-healthai_dev}
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./db/migrations:/docker-entrypoint-initdb.d  # Init schema sans Flyway
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-healthai}"]
      interval: 5s
      retries: 5

  etl:
    build: ./etl
    environment:
      DATABASE_URL: postgresql://${DB_USER:-healthai}:${DB_PASSWORD:-healthai_dev}@db:5432/${DB_NAME:-healthai}
      ETL_LOG_LEVEL: ${ETL_LOG_LEVEL:-INFO}
    volumes:
      - ./data/raw:/app/data/raw:ro          # Datasets en lecture seule
      - ./data/processed:/app/data/processed  # Exports apres nettoyage
    depends_on:
      db:
        condition: service_healthy

volumes:
  pgdata:
```

> **Note :** En phase dev, le schema est applique via les fichiers SQL montes dans `initdb.d`.
> En production, Flyway (cote Spring Boot) gere les migrations. Les memes fichiers SQL servent aux deux.

---

## Workflow de developpement Data

```
Etape 1                  Etape 2                  Etape 3
────────                 ────────                 ────────
Telecharger datasets     Ecrire les migrations    Developper les cleaners
dans data/raw/           SQL dans db/migrations/  un par un
                         + docker compose up db

       │                        │                        │
       v                        v                        v

Etape 4                  Etape 5                  Etape 6
────────                 ────────                 ────────
Ecrire le loader         Ecrire le generateur     Tester le pipeline
PostgreSQL               faker                     complet E2E
(insert/upsert)          (users business)          main.py → BDD OK ?
```

### Commandes quotidiennes

```bash
# Reset la base (supprime les donnees, re-applique le schema)
docker compose down -v && docker compose up db -d

# Lancer l'ETL sur une seule source (pour le dev iteratif)
python main.py --source nutrition
python main.py --source gym_members
python main.py --source exercisedb

# Lancer l'ETL complet
python main.py --all

# Verifier les donnees en base
docker compose exec db psql -U healthai -d healthai -c "SELECT COUNT(*) FROM nutrition_entries;"
docker compose exec db psql -U healthai -d healthai -c "SELECT * FROM etl_logs ORDER BY started_at DESC LIMIT 5;"
```

---

## .gitignore (section Data)

```gitignore
# Datasets bruts (trop volumineux, telecharges individuellement)
data/raw/*
!data/raw/.gitkeep

# Exports post-nettoyage
data/processed/*
!data/processed/.gitkeep

# Environnement Python
etl/__pycache__/
etl/.venv/
etl/*.egg-info/

# Variables d'environnement locales
.env
```

---

## Contrat d'interface avec le Backend (Dev 2-3)

L'ETL et l'API ne communiquent **jamais directement**. Le contrat est le schema PostgreSQL.

### Ce que Dev 1 (Data) fournit a Dev 2-3 (Backend) :

| Livrable | Format | Quand |
|---|---|---|
| Schema SQL (`V1__` a `V6__`) | Fichiers `.sql` dans `db/migrations/` | Avant que le Backend commence les entites JPA |
| Donnees de test en base | Pipeline ETL fonctionnel | Des que 2-3 sources sont chargees |
| Samples versionnes | CSV/JSON dans `data/samples/` | Immediatement |

### Ce que Dev 2-3 (Backend) fournit a Dev 1 (Data) :

| Livrable | Format | Quand |
|---|---|---|
| Validation du schema | Review des migrations SQL | Avant insertion en base |
| Endpoint declenchement ETL | `POST /api/admin/etl/run` | Sprint 2 |
