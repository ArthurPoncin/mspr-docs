# Plan Global - HealthAI Coach Backend

> **Projet :** MSPR TPRE501 - Backend HealthAI Coach
> **Equipe :** 4 developpeurs
> **Budget temps :** 19 heures par personne (76h total)
> **Epreuve :** 20min presentation + 30min questions jury
> **Derniere mise a jour :** 2026-03-16

---

## Table des matieres

1. [Vision du projet](#1-vision-du-projet)
2. [Stack technique](#2-stack-technique)
3. [Architecture](#3-architecture)
4. [Modele de donnees](#4-modele-de-donnees)
5. [Datasets et sources](#5-datasets-et-sources)
6. [Module ETL](#6-module-etl)
7. [API REST](#7-api-rest)
8. [Frontend et Dashboard](#8-frontend-et-dashboard)
9. [Securite](#9-securite)
10. [Accessibilite RGAA AA](#10-accessibilite-rgaa-aa)
11. [Infrastructure et deploiement](#11-infrastructure-et-deploiement)
12. [Repartition du travail](#12-repartition-du-travail)
13. [Planning previsionnel](#13-planning-previsionnel)
14. [Livrables](#14-livrables)
15. [Risques et mitigations](#15-risques-et-mitigations)
16. [Preparation soutenance](#16-preparation-soutenance)

---

## 1. Vision du projet

HealthAI Coach est une startup de sante connectee. Notre mission est de construire un **backend metier robuste** qui :

- **Centralise** des donnees heterogenes (nutrition, sport, biometrie) provenant de sources externes
- **Nettoie et valide** ces donnees via un pipeline ETL automatise
- **Expose** les donnees via une API REST securisee et documentee
- **Visualise** les indicateurs cles via un tableau de bord interactif accessible
- **Prepare** un socle de donnees fiable pour de futurs modeles d'Intelligence Artificielle

### Objectifs de qualite

| Critere | Cible |
|---|---|
| Qualite des donnees | 0 doublon, 0 valeur aberrante non traitee |
| Disponibilite API | Tous les endpoints CRUD fonctionnels |
| Documentation API | 100% des endpoints documentes OpenAPI |
| Accessibilite | Conformite RGAA niveau AA |
| Deploiement | Operationnel en < 30 minutes via Docker Compose |

---

## 2. Stack technique

### Choix et justifications

| Composant | Technologie | Justification |
|---|---|---|
| **ETL** | Python 3.12 + Pandas | Standard manipulation data, cite dans la webographie du sujet |
| **Orchestration** | `@Scheduled` Spring / cron | Suffisant pour le scope, zero surcharge d'infrastructure |
| **Base de donnees** | PostgreSQL 16 | Cite dans le sujet, standard industrie, excellent support JSON |
| **Migrations BDD** | Flyway | Integration native Spring Boot, versionning SQL explicite |
| **API REST** | Java 21 + Spring Boot 3 | Maitrise equipe, typage fort, ecosysteme structure Spring |
| **ORM** | Spring Data JPA (Hibernate) | Productivite CRUD, requetes derivees, support natif PostgreSQL |
| **Documentation API** | Springdoc OpenAPI 2 | Generation automatique Swagger UI, livrable obligatoire |
| **Authentification** | Spring Security + JWT | Standard securite, stateless, adapte REST |
| **Frontend** | React 18 (Vite) | Ecosysteme riche, controle total sur le rendu pour RGAA |
| **Graphiques** | Recharts | Librairie React accessible, support ARIA |
| **Conteneurisation** | Docker + Docker Compose | Exige par le sujet, reproductibilite garantie |

### Justification du choix Java pour l'API

1. **Competences existantes de l'equipe** — Choix pragmatique pour maximiser la productivite sur 19h. Miser sur ce qu'on maitrise pour livrer un produit robuste.
2. **Typage fort et cadre structure** — Spring Boot apporte injection de dependances, validation `@Valid`, gestion transactionnelle. Dans un contexte de donnees de sante, la fiabilite du traitement est critique.
3. **Ecosysteme industriel eprouve** — Spring Boot est le standard des SI d'entreprise en France (sante, banque, assurance). Demontrer sa maitrise dans un contexte data est un atout professionnel.

---

## 3. Architecture

### Vue d'ensemble

```
┌──────────────────────────────────────────────────────────────────┐
│                        Docker Compose                            │
│                                                                  │
│  ┌─────────────┐    ┌─────────────────┐    ┌─────────────────┐  │
│  │   Python     │    │  Spring Boot    │    │     React       │  │
│  │    ETL       │───>│     API         │<───│   Frontend +    │  │
│  │  Pandas      │    │  REST / JWT     │    │   Dashboard     │  │
│  │  + Cron      │    │  Springdoc      │    │   Recharts      │  │
│  └──────┬───────┘    └────────┬────────┘    └─────────────────┘  │
│         │                     │                                   │
│         v                     v                                   │
│        ┌───────────────────────────┐                             │
│        │       PostgreSQL 16       │                             │
│        │     (schema Flyway)       │                             │
│        └───────────────────────────┘                             │
└──────────────────────────────────────────────────────────────────┘
```

### Flux de donnees

```
Sources externes          Pipeline ETL              Base de donnees         Consommation
(CSV, JSON, XLSX)         (Python/Pandas)           (PostgreSQL)            (API + UI)

┌──────────┐         ┌──────────────────┐       ┌──────────────┐      ┌──────────────┐
│  Kaggle  │────────>│  1. Lecture       │──────>│  Tables      │<─────│  Spring Boot │
│  Datasets│         │  2. Validation    │       │  metier      │      │  API REST    │
└──────────┘         │  3. Nettoyage     │       ├──────────────┤      └──────┬───────┘
                     │  4. Transformation│       │  etl_logs    │             │
┌──────────┐         │  5. Chargement    │       │  (monitoring)│      ┌──────v───────┐
│  Donnees │────────>│  6. Logging       │──────>│              │      │    React     │
│  simulees│         └──────────────────┘       └──────────────┘      │  Dashboard   │
│  (faker) │                                                           └──────────────┘
└──────────┘
```

### Conteneurs Docker Compose

| Service | Image | Port | Dependance |
|---|---|---|---|
| `db` | postgres:16-alpine | 5432 | - |
| `etl` | python:3.12-slim (custom) | - | `db` |
| `api` | eclipse-temurin:21 (custom) | 8080 | `db` |
| `frontend` | node:20-alpine (build) + nginx | 3000 | `api` |

---

## 4. Modele de donnees

### MCD (Modele Conceptuel) - Vue simplifiee

> Le MCD complet sera documente separement au format Merise ou UML.

**Entites principales :**

- **User** — Utilisateur de la plateforme
- **NutritionEntry** — Entree alimentaire journaliere
- **ExerciseEntry** — Seance d'exercice
- **BiometricEntry** — Mesure biometrique (poids, sommeil, frequence cardiaque)
- **EtlLog** — Journal d'execution du pipeline
- **DataRecord** — Enregistrement brut avec statut de validation

### Schema relationnel previsionnel

```sql
-- Utilisateurs
users (
    id              BIGSERIAL PRIMARY KEY,
    email           VARCHAR(255) UNIQUE NOT NULL,
    username        VARCHAR(100) NOT NULL,
    password_hash   VARCHAR(255) NOT NULL,
    role            VARCHAR(20) DEFAULT 'USER',     -- USER, ADMIN
    is_premium      BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMP DEFAULT NOW(),
    last_activity   TIMESTAMP
);

-- Donnees nutritionnelles
nutrition_entries (
    id              BIGSERIAL PRIMARY KEY,
    user_id         BIGINT REFERENCES users(id),
    date            DATE NOT NULL,
    food_name       VARCHAR(255),
    calories        DECIMAL(8,2),
    protein_g       DECIMAL(8,2),
    carbs_g         DECIMAL(8,2),
    fat_g           DECIMAL(8,2),
    meal_type       VARCHAR(20),                    -- BREAKFAST, LUNCH, DINNER, SNACK
    source          VARCHAR(50),                    -- dataset d'origine
    status          VARCHAR(20) DEFAULT 'BRUT',     -- BRUT, NETTOYE, VALIDE, REJETE
    created_at      TIMESTAMP DEFAULT NOW()
);

-- Donnees exercice
exercise_entries (
    id              BIGSERIAL PRIMARY KEY,
    user_id         BIGINT REFERENCES users(id),
    date            DATE NOT NULL,
    activity_type   VARCHAR(100),
    duration_min    INTEGER,
    calories_burned DECIMAL(8,2),
    steps           INTEGER,
    distance_km     DECIMAL(8,2),
    heart_rate_avg  INTEGER,
    source          VARCHAR(50),
    status          VARCHAR(20) DEFAULT 'BRUT',
    created_at      TIMESTAMP DEFAULT NOW()
);

-- Donnees biometriques
biometric_entries (
    id              BIGSERIAL PRIMARY KEY,
    user_id         BIGINT REFERENCES users(id),
    date            DATE NOT NULL,
    weight_kg       DECIMAL(5,2),
    height_cm       DECIMAL(5,1),
    bmi             DECIMAL(5,2),
    fat_percentage  DECIMAL(5,2),
    heart_rate_rest INTEGER,
    heart_rate_avg  INTEGER,
    heart_rate_max  INTEGER,
    blood_pressure  VARCHAR(20),
    source          VARCHAR(50),
    status          VARCHAR(20) DEFAULT 'BRUT',
    created_at      TIMESTAMP DEFAULT NOW()
);

-- Catalogue exercices (referentiel depuis ExerciseDB)
exercises (
    id              BIGSERIAL PRIMARY KEY,
    external_id     VARCHAR(100) UNIQUE,            -- exerciseId ExerciseDB
    name            VARCHAR(255) NOT NULL,
    exercise_type   VARCHAR(50),                    -- STRENGTH, CARDIO, etc.
    body_parts      TEXT,                           -- JSON array ou CSV
    target_muscles  TEXT,                           -- JSON array ou CSV
    equipments      TEXT,                           -- JSON array ou CSV
    instructions    TEXT,
    source          VARCHAR(50) DEFAULT 'EXERCISEDB',
    created_at      TIMESTAMP DEFAULT NOW()
);

-- Journal ETL
etl_logs (
    id              BIGSERIAL PRIMARY KEY,
    source_name     VARCHAR(100) NOT NULL,
    started_at      TIMESTAMP NOT NULL,
    finished_at     TIMESTAMP,
    rows_read       INTEGER DEFAULT 0,
    rows_inserted   INTEGER DEFAULT 0,
    rows_rejected   INTEGER DEFAULT 0,
    error_count     INTEGER DEFAULT 0,
    status          VARCHAR(20),                    -- RUNNING, SUCCESS, FAILED
    details         TEXT
);
```

### Statuts de validation des donnees (workflow)

```
  Import ETL          Pipeline nettoyage        Action admin
      │                      │                       │
      v                      v                       v
   ┌──────┐            ┌──────────┐          ┌──────────┐
   │ BRUT │──────────> │ NETTOYE  │────────> │  VALIDE  │
   └──────┘            └──────────┘          └──────────┘
                             │
                             │               ┌──────────┐
                             └─────────────> │  REJETE  │
                                             └──────────┘
```

Seules les donnees au statut **VALIDE** alimentent les KPIs du dashboard.

---

## 5. Datasets et sources

### Sources selectionnees

> **Strategie :** Utiliser les 5 sources fournies par le sujet pour maximiser la couverture
> des 3 axes (nutrition, sport, biometrie) et demontrer la capacite a traiter des formats
> heterogenes (CSV plat, CSV profils complexes, JSON imbrique).
> Voir le document detaille : [`data/SOURCES.md`](data/SOURCES.md)

| # | Dataset | Domaine | Format | Volume | Lien |
|---|---|---|---|---|---|
| 1 | **Daily Food & Nutrition** | Nutrition | CSV | ~1000 entrees | [Kaggle](https://www.kaggle.com/datasets/adilshamim8/daily-food-and-nutrition-dataset) |
| 2 | **Diet Recommendations** | Nutrition + Profils sante | CSV | 1 000 profils | [Kaggle](https://www.kaggle.com/datasets/ziya07/diet-recommendations-dataset) |
| 3 | **ExerciseDB API** | Catalogue exercices | JSON (imbrique) | 11 000+ exercices | [GitHub](https://github.com/ExerciseDB/exercisedb-api/tree/main) |
| 4 | **Gym Members Exercise** | Profils utilisateurs + Biometrie | CSV | 973 echantillons | [Kaggle](https://www.kaggle.com/datasets/valakhorasani/gym-members-exercise-dataset) |
| 5 | **Fitness Tracker** | Activite quotidienne | CSV | ~1000 entrees | [Kaggle](https://www.kaggle.com/datasets/nadeemajeedch/fitness-tracker-dataset) |
| 6 | **Donnees simulees (faker)** | Business / Engagement | Genere | 500 utilisateurs | Script `fake_data_generator.py` |

### Mapping sources vers tables cibles

| Source | Table(s) cible | Colonnes exploitees |
|---|---|---|
| Daily Food & Nutrition | `nutrition_entries` | Food_Item, Calories, Protein, Carbs, Fat, Fiber, Sugars, Meal_Type |
| Diet Recommendations | `users` (enrichissement) | Patient_ID, Age, Gender, Weight, Height, BMI, Disease_Type, Diet_Recommendation |
| ExerciseDB API | `exercises` (table referentiel) | name, exerciseType, bodyParts, targetMuscles, equipments, instructions |
| Gym Members Exercise | `users` + `biometric_entries` + `exercise_entries` | Age, Gender, Weight, Height, Max_BPM, Avg_BPM, Resting_BPM, Calories_Burned, Workout_Type, Fat_Percentage, BMI |
| Fitness Tracker | `exercise_entries` (fusion avec Gym Members) | Steps, Calories_Burn, Active_Minutes, profils activite |
| Donnees simulees | `users` (complement) | email, username, is_premium, last_activity, engagement, satisfaction |

### Couverture des 3 axes du sujet

| Axe | Sources utilisees | Donnees cles |
|---|---|---|
| **Nutrition** | Daily Food & Nutrition + Diet Recommendations | Calories, macronutriments par repas, besoins dietetiques, recommandations |
| **Sport** | ExerciseDB + Gym Members + Fitness Tracker | Catalogue exercices (referentiel), seances effectuees, steps, calories brulees, duree |
| **Biometrie** | Gym Members + Diet Recommendations | Poids, taille, BMI, fat%, BPM repos/moy/max, pression arterielle |

### Cas de nettoyage remarquables (valeur pedagogique)

Ces cas seront mis en avant dans le rapport technique et la soutenance :

| Cas | Source(s) | Traitement |
|---|---|---|
| **Fusion de datasets quasi-identiques** | Gym Members + Fitness Tracker | Deduplication, normalisation des colonnes, detection des doublons inter-sources |
| **Aplatissement de JSON imbrique** | ExerciseDB | Transformation arrays (targetMuscles, equipments) en relations N-N ou colonnes concatenees |
| **Croisement de sources heterogenes** | Diet Reco + Gym Members | Rapprochement de profils utilisateurs par criteres demographiques (age, genre, poids) |
| **Generation de donnees complementaires** | Faker | Enrichissement des profils avec donnees business (engagement, premium, satisfaction) |

### Donnees simulees (faker)

Le script Python `fake_data_generator.py` produit :
- **500 utilisateurs** avec profils varies (age, genre, objectif, statut premium)
- **Historique d'activite** : dates de connexion, frequence d'utilisation
- **Metriques business** : taux d'engagement, score satisfaction (1-10)
- **Rattachement** : chaque entree des datasets Kaggle est rattachee a un utilisateur simule

---

## 6. Module ETL

### Pipeline par source

```
Pour chaque source de donnees :

1. EXTRACTION
   ├── Lecture fichier (CSV/JSON/XLSX) via Pandas
   └── Log : debut d'import dans etl_logs

2. VALIDATION STRUCTURE
   ├── Verification des colonnes attendues
   ├── Verification des types de donnees
   └── Rejet du fichier entier si structure invalide

3. NETTOYAGE
   ├── Suppression des doublons exacts
   ├── Traitement des valeurs manquantes (strategie par colonne)
   │   ├── Colonnes critiques (calories, date) : rejet de la ligne
   │   └── Colonnes secondaires (distance) : imputation mediane ou NaN
   ├── Detection des valeurs aberrantes (bornes metier)
   │   ├── calories < 0 ou > 10000 : rejet
   │   ├── heart_rate < 30 ou > 250 : rejet
   │   ├── sleep_hours < 0 ou > 24 : rejet
   │   └── weight_kg < 20 ou > 300 : rejet
   └── Normalisation des formats (dates ISO 8601, unites SI)

4. TRANSFORMATION
   ├── Rattachement a un user_id (mapping ou aleatoire pour les donnees Kaggle)
   ├── Calcul de champs derives (score nutrition, IMC)
   └── Assignation du statut NETTOYE

5. CHARGEMENT
   ├── Insertion en base PostgreSQL via psycopg2/SQLAlchemy
   └── Mode upsert pour eviter les doublons a la re-execution

6. LOGGING
   ├── Mise a jour etl_logs : lignes lues, inserees, rejetees
   └── Statut final : SUCCESS ou FAILED
```

### Regles de nettoyage metier

| Champ | Regle | Action si violation | Sources concernees |
|---|---|---|---|
| `calories` | 0 - 10 000 kcal | Rejet de la ligne | Daily Food, Gym Members, Fitness Tracker |
| `protein_g`, `carbs_g`, `fat_g` | >= 0 | Mise a 0 si negatif | Daily Food |
| `weight_kg` | 20 - 300 kg | Rejet de la ligne | Gym Members, Diet Reco |
| `height_cm` | 50 - 250 cm | Rejet de la ligne | Gym Members, Diet Reco |
| `bmi` | 10 - 80 | Recalcul depuis poids/taille | Gym Members, Diet Reco |
| `bpm_max` / `bpm_avg` / `bpm_rest` | 30 - 250 bpm | Rejet de la ligne | Gym Members |
| `fat_percentage` | 2 - 70 % | Rejet de la ligne | Gym Members |
| `session_duration` | 0 - 480 min | Rejet de la ligne | Gym Members, Fitness Tracker |
| `steps` | 0 - 100 000 | Rejet de la ligne | Fitness Tracker |
| `exercise_name` | Non vide, non null | Rejet de la ligne | ExerciseDB |
| `email` | Format valide, unique | Deduplication | Donnees simulees |

### Orchestration

L'execution du pipeline est declenchee de deux manieres :
- **Automatique** : tache `@Scheduled` dans Spring Boot (ou cron dans le conteneur ETL) — execution quotidienne
- **Manuelle** : endpoint `POST /api/admin/etl/run` declenche un import a la demande depuis l'interface admin

### Structure du code ETL

```
etl/
├── Dockerfile
├── requirements.txt          # pandas, psycopg2-binary, sqlalchemy, faker
├── config.py                 # connexion BDD, chemins fichiers
├── main.py                   # point d'entree, orchestration
├── extractors/
│   ├── csv_extractor.py
│   └── json_extractor.py
├── cleaners/
│   ├── nutrition_cleaner.py
│   ├── exercise_cleaner.py
│   └── biometric_cleaner.py
├── loaders/
│   └── postgres_loader.py
├── generators/
│   └── fake_data_generator.py
├── data/                     # datasets bruts (CSV/JSON)
│   ├── daily_food_nutrition.csv
│   ├── diet_recommendations.csv
│   ├── gym_members_exercise.csv
│   ├── fitness_tracker.csv
│   └── exercisedb/           # JSON exporte depuis le fork GitHub
│       └── exercises.json
└── logs/                     # logs fichier en complement de la BDD
```

---

## 7. API REST

### Endpoints

#### Authentification

| Methode | Endpoint | Description | Auth |
|---|---|---|---|
| POST | `/api/auth/register` | Inscription utilisateur | Non |
| POST | `/api/auth/login` | Connexion, retourne JWT | Non |
| POST | `/api/auth/refresh` | Rafraichir le token | JWT |

#### Utilisateurs

| Methode | Endpoint | Description | Auth |
|---|---|---|---|
| GET | `/api/users` | Liste des utilisateurs (pagine) | ADMIN |
| GET | `/api/users/{id}` | Detail utilisateur | USER (soi-meme) ou ADMIN |
| PUT | `/api/users/{id}` | Modifier profil | USER (soi-meme) ou ADMIN |
| DELETE | `/api/users/{id}` | Supprimer compte | ADMIN |

#### Nutrition

| Methode | Endpoint | Description | Auth |
|---|---|---|---|
| GET | `/api/nutrition` | Entrees nutrition (filtres: user, date, meal_type) | USER |
| GET | `/api/nutrition/{id}` | Detail entree | USER |
| POST | `/api/nutrition` | Ajouter entree | USER |
| PUT | `/api/nutrition/{id}` | Modifier entree | USER |
| DELETE | `/api/nutrition/{id}` | Supprimer entree | USER |

#### Exercice

| Methode | Endpoint | Description | Auth |
|---|---|---|---|
| GET | `/api/exercises` | Entrees exercice (filtres: user, date, type) | USER |
| GET | `/api/exercises/{id}` | Detail entree | USER |
| POST | `/api/exercises` | Ajouter entree | USER |
| PUT | `/api/exercises/{id}` | Modifier entree | USER |
| DELETE | `/api/exercises/{id}` | Supprimer entree | USER |

#### Biometrie

| Methode | Endpoint | Description | Auth |
|---|---|---|---|
| GET | `/api/biometrics` | Entrees biometriques (filtres: user, date) | USER |
| GET | `/api/biometrics/{id}` | Detail entree | USER |
| POST | `/api/biometrics` | Ajouter entree | USER |
| PUT | `/api/biometrics/{id}` | Modifier entree | USER |
| DELETE | `/api/biometrics/{id}` | Supprimer entree | USER |

#### Progression

| Methode | Endpoint | Description | Auth |
|---|---|---|---|
| GET | `/api/progression/{userId}` | Resume progression global | USER |
| GET | `/api/progression/{userId}/nutrition` | Evolution nutrition (hebdo/mensuel) | USER |
| GET | `/api/progression/{userId}/exercise` | Evolution exercice (hebdo/mensuel) | USER |
| GET | `/api/progression/{userId}/biometrics` | Evolution biometrie (hebdo/mensuel) | USER |

#### Administration

| Methode | Endpoint | Description | Auth |
|---|---|---|---|
| GET | `/api/admin/etl/logs` | Historique des imports ETL | ADMIN |
| POST | `/api/admin/etl/run` | Declencher un import ETL | ADMIN |
| GET | `/api/admin/data/anomalies` | Liste des donnees a statut NETTOYE ou REJETE | ADMIN |
| PATCH | `/api/admin/data/{table}/{id}/status` | Changer le statut d'un enregistrement (VALIDE/REJETE) | ADMIN |
| PATCH | `/api/admin/data/{table}/{id}` | Corriger une valeur manuellement | ADMIN |
| GET | `/api/admin/export/{table}` | Export CSV ou JSON (query param `format`) | ADMIN |

#### KPIs Dashboard

| Methode | Endpoint | Description | Auth |
|---|---|---|---|
| GET | `/api/dashboard/kpis/users` | Utilisateurs actifs, nouveaux, taux retention | ADMIN |
| GET | `/api/dashboard/kpis/nutrition` | Calories moy., repartition macros, score nutrition | ADMIN |
| GET | `/api/dashboard/kpis/fitness` | Heures exercice/semaine, pas moy., tendance | ADMIN |
| GET | `/api/dashboard/kpis/business` | Engagement, conversion premium, satisfaction | ADMIN |
| GET | `/api/dashboard/kpis/data-quality` | Taux erreur ETL, donnees en attente validation | ADMIN |

### Structure du code Spring Boot

```
src/main/java/com/healthai/coach/
├── HealthAiCoachApplication.java
├── config/
│   ├── SecurityConfig.java
│   ├── JwtConfig.java
│   └── CorsConfig.java
├── model/
│   ├── User.java
│   ├── NutritionEntry.java
│   ├── ExerciseEntry.java
│   ├── BiometricEntry.java
│   ├── EtlLog.java
│   └── enums/
│       ├── DataStatus.java          # BRUT, NETTOYE, VALIDE, REJETE
│       ├── MealType.java
│       └── Role.java
├── repository/
│   ├── UserRepository.java
│   ├── NutritionRepository.java
│   ├── ExerciseRepository.java
│   ├── BiometricRepository.java
│   └── EtlLogRepository.java
├── service/
│   ├── AuthService.java
│   ├── UserService.java
│   ├── NutritionService.java
│   ├── ExerciseService.java
│   ├── BiometricService.java
│   ├── ProgressionService.java
│   ├── DashboardService.java
│   ├── AdminService.java
│   └── ExportService.java
├── controller/
│   ├── AuthController.java
│   ├── UserController.java
│   ├── NutritionController.java
│   ├── ExerciseController.java
│   ├── BiometricController.java
│   ├── ProgressionController.java
│   ├── DashboardController.java
│   └── AdminController.java
├── dto/
│   ├── request/
│   └── response/
├── security/
│   ├── JwtTokenProvider.java
│   ├── JwtAuthenticationFilter.java
│   └── UserDetailsServiceImpl.java
└── exception/
    ├── GlobalExceptionHandler.java
    └── ResourceNotFoundException.java
```

---

## 8. Frontend et Dashboard

### Pages de l'application

#### Interface d'administration

| Page | Fonctionnalites |
|---|---|
| **Login** | Formulaire connexion admin |
| **Dashboard ETL** | Derniers imports, lignes traitees/rejetees, taux d'erreur, graphique timeline |
| **Gestion anomalies** | Tableau paginable des donnees NETTOYE/REJETE, correction inline, boutons Valider/Rejeter |
| **Export donnees** | Selection table + format (CSV/JSON) + filtres, bouton telecharger |
| **Gestion utilisateurs** | Liste utilisateurs, detail, suppression |

#### Tableau de bord analytique

| Onglet | KPIs affiches | Type de graphique |
|---|---|---|
| **Vue globale** | Utilisateurs actifs, score sante moyen, alertes | Cartes chiffres + sparklines |
| **Nutrition** | Calories moy/jour, repartition macros, top aliments | Bar chart + pie chart |
| **Fitness** | Heures exercice/sem, pas/jour, evolution | Line chart + bar chart |
| **Business** | Taux engagement, conversion premium, satisfaction | Gauges + line chart tendance |
| **Qualite donnees** | Donnees en attente, taux erreur par source, historique ETL | Stacked bar + table |

### Structure du code React

```
frontend/
├── Dockerfile
├── nginx.conf
├── package.json
├── src/
│   ├── main.jsx
│   ├── App.jsx
│   ├── api/
│   │   └── client.js                # Axios instance avec intercepteur JWT
│   ├── auth/
│   │   ├── AuthContext.jsx
│   │   └── ProtectedRoute.jsx
│   ├── pages/
│   │   ├── LoginPage.jsx
│   │   ├── DashboardPage.jsx
│   │   ├── AnomaliesPage.jsx
│   │   ├── ExportPage.jsx
│   │   └── UsersPage.jsx
│   ├── components/
│   │   ├── layout/
│   │   │   ├── Sidebar.jsx
│   │   │   └── Header.jsx
│   │   ├── dashboard/
│   │   │   ├── KpiCard.jsx
│   │   │   ├── NutritionChart.jsx
│   │   │   ├── FitnessChart.jsx
│   │   │   ├── BusinessChart.jsx
│   │   │   └── DataQualityChart.jsx
│   │   ├── admin/
│   │   │   ├── AnomalyTable.jsx
│   │   │   ├── EtlStatusPanel.jsx
│   │   │   └── ExportForm.jsx
│   │   └── ui/
│   │       ├── Button.jsx
│   │       ├── Table.jsx
│   │       ├── Modal.jsx
│   │       └── Alert.jsx
│   └── styles/
│       └── globals.css
```

---

## 9. Securite

### Authentification JWT

```
Client                          API Spring Boot
  │                                    │
  │  POST /api/auth/login              │
  │  { email, password }               │
  │───────────────────────────────────>│
  │                                    │  Verification credentials
  │  200 { accessToken, refreshToken } │  Generation JWT
  │<───────────────────────────────────│
  │                                    │
  │  GET /api/nutrition                │
  │  Authorization: Bearer <token>     │
  │───────────────────────────────────>│
  │                                    │  Validation JWT
  │  200 { data: [...] }               │  Extraction role
  │<───────────────────────────────────│
```

### Regles de securite

| Regle | Implementation |
|---|---|
| Mots de passe hashes | BCrypt via Spring Security |
| Tokens signes | HMAC-SHA256 (secret en variable d'environnement) |
| Expiration access token | 15 minutes |
| Expiration refresh token | 7 jours |
| Protection CSRF | Desactivee (API stateless) |
| CORS | Restreint au domaine frontend |
| Roles | USER (ses propres donnees), ADMIN (tout) |

---

## 10. Accessibilite RGAA AA

> **CRITIQUE** : Le respect RGAA AA est un critere determinant de l'evaluation.

### Checklist d'implementation

| Critere RGAA | Implementation concrete |
|---|---|
| **Contrastes** (1.4.3) | Ratio minimum 4.5:1 texte normal, 3:1 grand texte. Verifier avec axe DevTools. |
| **Navigation clavier** (2.1.1) | Tous les elements interactifs atteignables au clavier. Focus visible sur chaque element. |
| **Alternatives textuelles** (1.1.1) | Attribut `alt` sur toutes les images. `aria-label` sur tous les graphiques Recharts. |
| **Structure semantique** (1.3.1) | Utilisation correcte de `<main>`, `<nav>`, `<header>`, `<h1>`-`<h6>`, `<table>`. |
| **Formulaires** (1.3.1, 3.3.2) | `<label>` associe a chaque `<input>`. Messages d'erreur explicites lies au champ. |
| **Tableaux de donnees** (1.3.1) | `<th scope="col/row">`, `<caption>` descriptive. |
| **Graphiques** (1.1.1) | Description textuelle alternative pour chaque graphique. Donnees aussi disponibles sous forme de tableau. |
| **Skip navigation** (2.4.1) | Lien "Aller au contenu principal" en haut de chaque page. |
| **Langue de la page** (3.1.1) | `<html lang="fr">` |
| **Zoom** (1.4.4) | Interface lisible a 200% de zoom, pas de scroll horizontal. |

### Outils de verification

- **axe DevTools** (extension navigateur) — audit automatise
- **Lighthouse** (onglet Accessibility) — score cible > 90
- **Navigation clavier manuelle** — test Tab / Shift+Tab / Enter / Escape sur chaque page
- **NVDA ou VoiceOver** — test rapide avec lecteur d'ecran si possible

---

## 11. Infrastructure et deploiement

### Docker Compose

```yaml
# docker-compose.yml (structure cible)
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: healthai
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 5s
      retries: 5

  etl:
    build: ./etl
    environment:
      DATABASE_URL: postgresql://${DB_USER}:${DB_PASSWORD}@db:5432/healthai
    depends_on:
      db:
        condition: service_healthy

  api:
    build: ./api
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/healthai
      SPRING_DATASOURCE_USERNAME: ${DB_USER}
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD}
      JWT_SECRET: ${JWT_SECRET}
    ports:
      - "8080:8080"
    depends_on:
      db:
        condition: service_healthy
      etl:
        condition: service_completed_successfully

  frontend:
    build: ./frontend
    ports:
      - "3000:80"
    depends_on:
      - api

volumes:
  pgdata:
```

### Procedure de deploiement (< 30 minutes)

```bash
# 1. Cloner le repository
git clone https://github.com/[org]/healthai-coach.git
cd healthai-coach

# 2. Configurer les variables d'environnement
cp .env.example .env
# Editer .env avec les credentials souhaites

# 3. Lancer l'ensemble
docker compose up --build -d

# 4. Verifier
docker compose ps                           # tous les services UP
curl http://localhost:8080/api/health        # API OK
open http://localhost:3000                   # Frontend accessible
open http://localhost:8080/swagger-ui.html   # Documentation API
```

---

## 12. Repartition du travail

### Affectation par personne

| Personne | Role | Perimetre principal | Perimetre secondaire |
|---|---|---|---|
| **Dev 1** | Data Engineer | ETL Python/Pandas, selection et nettoyage datasets, fake data (faker), table `etl_logs` | Documentation sources de donnees |
| **Dev 2** | Backend Lead | Spring Boot : entites JPA, repositories, services CRUD, endpoints Progression + KPIs, Flyway migrations | Documentation modele de donnees (MCD) |
| **Dev 3** | Backend / DevOps | Spring Security JWT, endpoints Admin (validation, correction, export), Docker Compose, guide deploiement | Rapport technique |
| **Dev 4** | Frontend Lead | React : admin (anomalies, validation, export), dashboard (Recharts), RGAA AA, integration API | Support de presentation |

### Livrables transversaux (repartis sur l'equipe)

| Livrable | Responsable principal | Contributeurs |
|---|---|---|
| Modele de donnees (MCD/MLD) | Dev 2 | Dev 1 (validation colonnes data) |
| Scripts SQL / migrations Flyway | Dev 2 | Dev 3 (review) |
| Rapport technique (5-8 pages) | Dev 3 | Tous |
| Support de presentation | Dev 4 | Tous |
| Tests integration Docker | Dev 3 | Tous (validation sur machine vierge) |

---

## 13. Planning previsionnel

### Phase 1 — Fondations (heures 1-5)

| Tache | Qui | Duree estimee |
|---|---|---|
| Modelisation BDD (MCD/MLD), validation en equipe | Dev 2 + tous | 2h |
| Setup projet Spring Boot + Flyway + premier schema | Dev 2 | 3h |
| Setup projet ETL Python, premiers scripts de lecture | Dev 1 | 3h |
| Setup Spring Security + JWT | Dev 3 | 3h |
| Setup projet React (Vite), layout principal, routing | Dev 4 | 3h |
| Setup Docker Compose (db + structure) | Dev 3 | 2h |

### Phase 2 — Developpement core (heures 5-14)

| Tache | Qui | Duree estimee |
|---|---|---|
| Pipeline ETL complet : extract, clean, load, logs | Dev 1 | 7h |
| Generation donnees simulees (faker) | Dev 1 | 2h |
| CRUD complet (users, nutrition, exercise, biometrics) | Dev 2 | 6h |
| Endpoints progression et KPIs dashboard | Dev 2 | 3h |
| Endpoints admin (anomalies, validation, export) | Dev 3 | 5h |
| Docker Compose complet + healthchecks | Dev 3 | 2h |
| Page login + dashboard KPIs (Recharts) | Dev 4 | 5h |
| Page admin anomalies (tableau, correction, validation) | Dev 4 | 4h |

### Phase 3 — Integration et finitions (heures 14-17)

| Tache | Qui | Duree estimee |
|---|---|---|
| Integration frontend-API (tous les flux) | Dev 4 + Dev 2 | 2h |
| Accessibilite RGAA AA (audit + corrections) | Dev 4 | 3h |
| Tests E2E : pipeline ETL > API > Dashboard | Dev 1 + Dev 3 | 2h |
| Test deploiement Docker sur machine vierge | Dev 3 | 1h |

### Phase 4 — Documentation et preparation (heures 17-19)

| Tache | Qui | Duree estimee |
|---|---|---|
| Rapport technique (5-8 pages) | Dev 3 + tous | 3h |
| Documentation API (verification Swagger) | Dev 2 | 0.5h |
| Support de presentation | Dev 4 + tous | 2h |
| Repetition soutenance (20 min) | Tous | 1h |

---

## 14. Livrables

### Checklist finale

| # | Livrable | Format | Critere de validation |
|---|---|---|---|
| 1 | Inventaire des sources de donnees | Section dans rapport ou doc separee | Au moins 2 sources justifiees |
| 2 | Diagramme des flux de donnees | Image / schema dans le rapport | Flux complet source > ETL > BDD > API > UI |
| 3 | Code ETL | Repository Git, commente, avec logs | Pipeline reproductible, logs en base |
| 4 | Jeux de donnees nettoyes | Fichiers exports ou donnees en base | Zero doublon, zero valeur hors bornes |
| 5 | Modele de donnees | MCD + MLD (Merise ou UML) | Coherent avec le schema SQL |
| 6 | Scripts SQL / migrations | Flyway (V1__*.sql, V2__*.sql ...) | Execution automatique au demarrage |
| 7 | API REST fonctionnelle | Spring Boot deployable | Tous les CRUD + KPIs operationnels |
| 8 | Documentation OpenAPI | Swagger UI auto-generee | 100% des endpoints documentes |
| 9 | Interface admin | React | Visualisation flux, correction anomalies, export |
| 10 | Tableau de bord | React + Recharts | KPIs nutrition, fitness, business, qualite donnees |
| 11 | Conformite RGAA AA | Audit Lighthouse / axe | Score > 90, navigation clavier OK |
| 12 | Rapport technique | PDF, 5-8 pages | Contexte, choix, difficultes, perspectives |
| 13 | Docker Compose | `docker-compose.yml` + `.env.example` | Deploiement < 30 min sur machine vierge |
| 14 | Guide de deploiement | README ou doc dediee | Etapes claires et reproductibles |
| 15 | Support de presentation | Slides | 20 min de presentation structuree |

---

## 15. Risques et mitigations

| # | Risque | Probabilite | Impact | Mitigation |
|---|---|---|---|---|
| 1 | **Depassement du budget horaire** | Haute | Critique | Prioriser : ETL + API CRUD + Dashboard basique avant les extras. Couper les fonctionnalites non essentielles. |
| 2 | **Incompatibilite des datasets Kaggle** | Moyenne | Eleve | Identifier et telecharger les datasets des la premiere heure. Valider la structure avant de coder. |
| 3 | **Difficulte integration Docker multi-services** | Moyenne | Eleve | Dev 3 setup Docker Compose des le jour 1 avec un docker-compose minimal (db + api). Ajouter les services un par un. |
| 4 | **Non-conformite RGAA AA** | Haute | Critique | Integrer les bonnes pratiques ARIA des le debut du dev React (pas en derniere minute). Audit axe apres chaque page. |
| 5 | **Complexite Spring Security JWT** | Moyenne | Moyen | Utiliser un template Spring Security eprouve. Cas de repli : basic auth si JWT trop complexe. |
| 6 | **Communication ETL Python <> API Java** | Faible | Moyen | Les deux services communiquent via PostgreSQL uniquement (pas d'appels API entre eux). Decouplage total. |
| 7 | **Manque de temps pour la documentation** | Haute | Eleve | Rediger le rapport en continu (pas a la fin). Chaque dev documente son perimetre au fil de l'eau. |

### Priorite en cas de manque de temps (ordre de sacrifice)

Si le temps vient a manquer, sacrifier dans cet ordre (du moins critique au plus critique) :

1. ~~KPIs business (engagement, conversion)~~ — peuvent etre simplifies
2. ~~Endpoint refresh token~~ — le login simple suffit
3. ~~Export JSON (garder CSV uniquement)~~ — moins de code
4. **Jamais sacrifier** : CRUD de base, documentation OpenAPI, RGAA AA, Docker Compose, rapport

---

## 16. Preparation soutenance

### Structure de la presentation (20 minutes)

| Temps | Section | Contenu | Presentateur suggere |
|---|---|---|---|
| 0-3 min | **Contexte** | Problematique HealthAI Coach, enjeux data sante | Dev 1 |
| 3-7 min | **Architecture et choix techniques** | Stack, justification Java, schema architecture | Dev 2 |
| 7-10 min | **Pipeline ETL** | Demo : import d'un dataset, logs, donnees nettoyees | Dev 1 |
| 10-14 min | **API et securite** | Demo : Swagger UI, authentification JWT, CRUD | Dev 3 |
| 14-18 min | **Interface et dashboard** | Demo live : admin, validation donnees, KPIs | Dev 4 |
| 18-20 min | **Bilan** | Difficultes, perspectives IA, retour d'experience | Dev 2 |

### Questions probables du jury (a preparer)

| Theme | Questions anticipees |
|---|---|
| **Choix techniques** | Pourquoi Java plutot que Python pour l'API ? Comment justifiez-vous deux langages ? |
| **Data quality** | Comment garantissez-vous la qualite des donnees ? Quelles regles metier avez-vous definies ? |
| **Scalabilite** | Comment votre architecture evoluerait avec 100x plus de donnees ? |
| **Securite** | Comment gerez-vous l'expiration des tokens ? Quels risques OWASP avez-vous adresses ? |
| **RGAA** | Comment avez-vous teste l'accessibilite ? Montrez-moi la navigation clavier. |
| **Perspectives IA** | Comment vos donnees seraient exploitees par un modele ML ? Quels features pour quel modele ? |
| **Methodologie** | Comment vous etes-vous organises a 4 ? Quel outil de gestion de projet ? |
| **Difficultes** | Quelle a ete la plus grande difficulte technique ? Comment l'avez-vous resolue ? |

---

## Annexes

### Variables d'environnement (.env.example)

```env
# Base de donnees
DB_USER=healthai
DB_PASSWORD=changeme
POSTGRES_DB=healthai

# JWT
JWT_SECRET=changeme-with-a-long-random-string
JWT_EXPIRATION_MS=900000
JWT_REFRESH_EXPIRATION_MS=604800000

# ETL
ETL_DATA_PATH=/app/data
ETL_LOG_LEVEL=INFO
```

### Liens utiles

- [Spring Boot Reference](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Springdoc OpenAPI](https://springdoc.org/)
- [Flyway Documentation](https://documentation.red-gate.com/fd)
- [Recharts](https://recharts.org/)
- [RGAA 4.1 Referentiel](https://accessibilite.numerique.gouv.fr/methode/criteres-et-tests/)
- [axe DevTools](https://www.deque.com/axe/devtools/)
