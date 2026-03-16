# Inventaire des Sources de Donnees

> **Livrable :** Ce document repond a l'exigence de l'inventaire des sources de donnees (livrable 1 du cahier des charges).
> **Derniere mise a jour :** 2026-03-16

---

## Vue d'ensemble

Le projet exploite **5 sources de donnees externes** fournies par le cahier des charges, completees par **1 generateur de donnees simulees**. Elles couvrent les 3 axes metier de HealthAI Coach : nutrition, sport et biometrie.

```
                    ┌─────────────────────────────────┐
                    │       SOURCES DE DONNEES         │
                    └─────────────────────────────────┘
                                   │
          ┌────────────────────────┼────────────────────────┐
          │                        │                        │
    ┌─────v──────┐          ┌──────v──────┐          ┌──────v──────┐
    │  NUTRITION  │          │    SPORT    │          │  BIOMETRIE  │
    └─────┬──────┘          └──────┬──────┘          └──────┬──────┘
          │                        │                        │
   ┌──────┴──────┐          ┌──────┴──────┐          ┌──────┴──────┐
   │ Daily Food  │          │ ExerciseDB  │          │ Gym Members │
   │ & Nutrition │          │ (referentiel│          │  Exercise   │
   │   (CSV)     │          │   JSON)     │          │   (CSV)     │
   └─────────────┘          ├─────────────┤          └─────────────┘
   ┌─────────────┐          │ Gym Members │          ┌─────────────┐
   │    Diet     │          │  (seances)  │          │    Diet     │
   │ Recomm.     │          ├─────────────┤          │  Recomm.    │
   │   (CSV)     │          │  Fitness    │          │ (profils)   │
   └─────────────┘          │  Tracker    │          └─────────────┘
                            │   (CSV)     │
                            └─────────────┘

                    ┌─────────────────────────────────┐
                    │  + Donnees simulees (faker)      │
                    │    Profils, engagement, premium   │
                    └─────────────────────────────────┘
```

---

## Source 1 : Daily Food & Nutrition Dataset

| Propriete | Valeur |
|---|---|
| **Lien** | https://www.kaggle.com/datasets/adilshamim8/daily-food-and-nutrition-dataset |
| **Format** | CSV |
| **Volume** | ~1 000 entrees |
| **Licence** | Kaggle (usage educatif) |
| **Axe couvert** | Nutrition |
| **Table cible** | `nutrition_entries` |

### Colonnes attendues

| Colonne | Type | Description | Utilisation |
|---|---|---|---|
| `Food_Item` | string | Nom de l'aliment | `food_name` |
| `Category` | string | Categorie alimentaire (fruit, cereal, meat...) | Enrichissement / filtrage |
| `Calories` | float | Apport calorique (kcal) | `calories` |
| `Protein(g)` | float | Proteines en grammes | `protein_g` |
| `Carbohydrates(g)` | float | Glucides en grammes | `carbs_g` |
| `Fat(g)` | float | Lipides en grammes | `fat_g` |
| `Fiber(g)` | float | Fibres en grammes | Enrichissement |
| `Sugars(g)` | float | Sucres en grammes | Enrichissement |
| `Sodium(mg)` | float | Sodium en milligrammes | Enrichissement |
| `Cholesterol(mg)` | float | Cholesterol en milligrammes | Enrichissement |
| `Meal_Type` | string | Type de repas (Breakfast, Lunch, Dinner, Snack) | `meal_type` |
| `Water_Intake(liters)` | float | Consommation d'eau en litres | Enrichissement |

### Regles de nettoyage specifiques

| Regle | Condition | Action |
|---|---|---|
| Calories valides | `Calories` < 0 ou > 10 000 | Rejet de la ligne |
| Macronutriments positifs | `Protein`, `Carbs`, `Fat` < 0 | Mise a 0 |
| Coherence macros/calories | \|Protein*4 + Carbs*4 + Fat*9 - Calories\| > 200 | Flag anomalie (ne pas rejeter, tolerance sur les fibres/alcool) |
| Meal_Type valide | Valeur hors {Breakfast, Lunch, Dinner, Snack} | Assignation "Other" |
| Food_Item non vide | Champ vide ou null | Rejet de la ligne |
| Doublons | Meme Food_Item + meme Meal_Type + memes valeurs nutritionnelles | Deduplication |

### Justification du choix

Cette source fournit les donnees nutritionnelles de base essentielles au domaine de HealthAI Coach. Elle permet de calculer les KPIs nutrition (calories moyennes/jour, repartition macronutriments, score nutritionnel). La granularite par type de repas (`Meal_Type`) permet des analyses intra-journalieres.

---

## Source 2 : Diet Recommendations Dataset

| Propriete | Valeur |
|---|---|
| **Lien** | https://www.kaggle.com/datasets/ziya07/diet-recommendations-dataset |
| **Format** | CSV |
| **Volume** | 1 000 profils patients |
| **Licence** | Kaggle (usage educatif) |
| **Axe couvert** | Nutrition + Biometrie (profils) |
| **Tables cibles** | `users` (enrichissement), `biometric_entries` |

### Colonnes attendues

| Colonne | Type | Description | Utilisation |
|---|---|---|---|
| `Patient_ID` | int | Identifiant unique | Cle de rapprochement |
| `Age` | int | Age en annees | `users.age` |
| `Gender` | string | Genre (Male/Female) | `users.gender` |
| `Weight(kg)` | float | Poids en kg | `biometric_entries.weight_kg` |
| `Height(cm)` | float | Taille en cm | `biometric_entries.height_cm` |
| `BMI` | float | Indice de masse corporelle | `biometric_entries.bmi` (verifie par recalcul) |
| `Disease_Type` | string | Type de pathologie | Enrichissement profil |
| `Physical_Activity_Level` | string | Niveau d'activite | Enrichissement profil |
| `Daily_Caloric_Intake` | float | Apport calorique quotidien | Croisement avec nutrition |
| `Blood_Pressure` | string | Pression arterielle (systolique/diastolique) | `biometric_entries.blood_pressure` |
| `Diet_Recommendation` | string | Recommandation dietetique | Enrichissement / futur axe IA |

### Regles de nettoyage specifiques

| Regle | Condition | Action |
|---|---|---|
| Age valide | `Age` < 10 ou > 120 | Rejet de la ligne |
| Poids valide | `Weight` < 20 ou > 300 | Rejet de la ligne |
| Taille valide | `Height` < 50 ou > 250 | Rejet de la ligne |
| BMI coherent | \|BMI - Weight/(Height/100)^2\| > 2 | Recalcul du BMI depuis poids/taille |
| Genre normalise | Valeur hors {Male, Female, Other} | Normalisation (M -> Male, F -> Female) |
| Blood Pressure format | Ne correspond pas a "XXX/YYY" | Flag anomalie |

### Justification du choix

Cette source apporte la dimension **profils de sante** absente des autres datasets. Les recommandations dietetiques et les pathologies permettent de simuler des parcours utilisateurs realistes (perte de poids, diabete, hypertension). Le `BMI` et la `Blood_Pressure` alimentent directement les donnees biometriques.

---

## Source 3 : ExerciseDB API Repository

| Propriete | Valeur |
|---|---|
| **Lien** | https://github.com/ExerciseDB/exercisedb-api/tree/main |
| **Format** | JSON (structure imbriquee avec arrays) |
| **Volume** | 11 000+ exercices |
| **Licence** | AGPL-3.0 |
| **Axe couvert** | Sport (referentiel) |
| **Table cible** | `exercises` |

### Structure JSON par exercice

```json
{
  "exerciseId": "exr_41n2hZZdH9uyYFGZ",
  "name": "Bench Press",
  "exerciseType": "STRENGTH",
  "bodyParts": ["CHEST"],
  "targetMuscles": ["PECTORALIS_MAJOR"],
  "secondaryMuscles": ["ANTERIOR_DELTOID", "TRICEPS"],
  "equipments": ["BARBELL", "BENCH"],
  "instructions": ["Lie on bench...", "Lower bar to chest...", "Press up..."],
  "overview": "The bench press is a compound exercise...",
  "exerciseTips": ["Keep feet flat...", "Maintain arch..."],
  "gender": "MALE",
  "imageUrl": "...",
  "videoUrl": "..."
}
```

### Colonnes extraites pour la table `exercises`

| Champ JSON | Transformation | Colonne cible |
|---|---|---|
| `exerciseId` | Direct | `external_id` |
| `name` | Direct | `name` |
| `exerciseType` | Direct | `exercise_type` |
| `bodyParts` | Array -> texte separe par virgules | `body_parts` |
| `targetMuscles` | Array -> texte separe par virgules | `target_muscles` |
| `equipments` | Array -> texte separe par virgules | `equipments` |
| `instructions` | Array -> texte concatene avec sauts de ligne | `instructions` |

> Les champs `imageUrl`, `videoUrl`, `exerciseTips`, `variations`, `relatedExerciseIds` ne sont **pas** importes (hors scope, poids en base inutile).

### Regles de nettoyage specifiques

| Regle | Condition | Action |
|---|---|---|
| Nom non vide | `name` vide ou null | Rejet de l'exercice |
| ID unique | `exerciseId` duplique | Garder la premiere occurrence |
| Type valide | `exerciseType` vide | Assignation "UNKNOWN" |
| Arrays non vides | `bodyParts` ou `targetMuscles` vide | Accepte (pas critique) |

### Methode d'extraction

Le repository GitHub ne contient que le README et la licence. Les donnees sont servies par l'API via RapidAPI. Methode recommandee :
1. **Fork** du repository sur le compte GitHub de l'equipe (demande par le sujet)
2. **Extraction** via l'API RapidAPI (plan gratuit) ou via un scraping du cache public
3. **Export** en fichier `exercises.json` local dans `data/raw/exercisedb/`
4. Le pipeline ETL lit ce fichier JSON local (pas d'appel API a chaque execution)

### Justification du choix

ExerciseDB est la seule source de type **JSON imbrique** (arrays de strings), ce qui enrichit le pipeline ETL avec un cas de transformation non trivial (aplatissement). Elle fournit un **catalogue de reference** (11 000+ exercices) qui peut etre lie aux seances des utilisateurs pour des analyses detaillees (muscles travailles, equipement utilise).

---

## Source 4 : Gym Members Exercise Dataset

| Propriete | Valeur |
|---|---|
| **Lien** | https://www.kaggle.com/datasets/valakhorasani/gym-members-exercise-dataset |
| **Format** | CSV |
| **Volume** | 973 echantillons |
| **Licence** | Kaggle (usage educatif) |
| **Axes couverts** | Sport + Biometrie + Profils |
| **Tables cibles** | `users`, `exercise_entries`, `biometric_entries` |

### Colonnes attendues

| Colonne | Type | Description | Table cible |
|---|---|---|---|
| `Age` | int | Age du membre | `users` |
| `Gender` | string | Genre | `users` |
| `Weight (kg)` | float | Poids | `biometric_entries` |
| `Height (m)` | float | Taille en metres | `biometric_entries` (convertir en cm) |
| `Max_BPM` | int | Frequence cardiaque maximale | `biometric_entries` |
| `Avg_BPM` | int | Frequence cardiaque moyenne | `biometric_entries` |
| `Resting_BPM` | int | Frequence cardiaque au repos | `biometric_entries` |
| `Session_Duration (hours)` | float | Duree de la seance | `exercise_entries` |
| `Calories_Burned` | float | Calories brulees par seance | `exercise_entries` |
| `Workout_Type` | string | Type d'exercice (Yoga, HIIT, Cardio, Strength) | `exercise_entries` |
| `Fat_Percentage` | float | Pourcentage de graisse corporelle | `biometric_entries` |
| `Water_Intake (liters)` | float | Consommation d'eau | Enrichissement |
| `Workout_Frequency (days/week)` | int | Frequence d'entrainement | Enrichissement |
| `Experience_Level` | int | Niveau (1-3) | `users` (profil) |
| `BMI` | float | Indice de masse corporelle | `biometric_entries` |

### Regles de nettoyage specifiques

| Regle | Condition | Action |
|---|---|---|
| Age valide | `Age` < 10 ou > 100 | Rejet |
| Poids valide | `Weight` < 20 ou > 300 | Rejet |
| Taille (conversion) | `Height` en metres | Conversion en cm (`* 100`) |
| BPM coherent | `Resting_BPM` > `Avg_BPM` ou `Avg_BPM` > `Max_BPM` | Flag anomalie |
| BPM dans les bornes | Tout BPM < 30 ou > 250 | Rejet |
| Fat_Percentage valide | < 2% ou > 70% | Rejet |
| Session_Duration valide | <= 0 ou > 8h | Rejet |
| Workout_Type valide | Valeur hors {Yoga, HIIT, Cardio, Strength} | Assignation "Other" |

### Justification du choix

Dataset le plus riche du lot : il alimente **3 tables** a lui seul (users, exercise, biometric). Les donnees BPM (repos/moyen/max) et fat% apportent la couverture biometrique exigee par le sujet. Le champ `Experience_Level` permet de segmenter les utilisateurs.

---

## Source 5 : Fitness Tracker Dataset

| Propriete | Valeur |
|---|---|
| **Lien** | https://www.kaggle.com/datasets/nadeemajeedch/fitness-tracker-dataset |
| **Format** | CSV |
| **Volume** | ~1 000 entrees |
| **Licence** | Kaggle (usage educatif) |
| **Axe couvert** | Sport (activite quotidienne) |
| **Table cible** | `exercise_entries` (fusion avec source 4) |

### Colonnes attendues

> Ce dataset est structurellement tres proche du Gym Members Exercise Dataset.
> Les colonnes exactes seront confirmees apres telechargement, mais on attend :

| Colonne | Type | Description |
|---|---|---|
| `Steps` | int | Nombre de pas quotidiens |
| `Calories_Burn` | float | Calories brulees |
| `Active_Minutes` | int | Minutes d'activite |
| `Distance (km)` | float | Distance parcourue |
| `Workout_Type` | string | Type d'activite |
| Profils demographiques | varies | Age, Genre, Poids, Taille |

### Strategie de fusion avec Source 4

Les datasets 4 et 5 etant quasi-identiques en structure :
1. **Normalisation** des noms de colonnes (meme schema cible)
2. **Ajout d'un champ `source`** pour tracer l'origine (`GYM_MEMBERS` vs `FITNESS_TRACKER`)
3. **Deduplication inter-sources** : si un enregistrement est identique (memes valeurs demographiques + memes metriques), on ne garde qu'un exemplaire
4. **Concatenation** des enregistrements uniques pour augmenter le volume total

Ce cas de fusion est un **excellent exemple de nettoyage** a presenter au jury.

### Justification du choix

Apporte le `Steps` (nombre de pas) et la `Distance` absents du dataset Gym Members. La fusion des deux sources augmente le volume de donnees et constitue un cas concret de deduplication inter-sources.

---

## Source 6 : Donnees simulees (faker)

| Propriete | Valeur |
|---|---|
| **Format** | Genere par script Python |
| **Volume** | 500 utilisateurs + metriques associees |
| **Axe couvert** | Business / Engagement |
| **Table cible** | `users` (complement) |

### Donnees generees

| Donnee | Type | Plage | Description |
|---|---|---|---|
| `email` | string | Format valide | Adresse email unique |
| `username` | string | Prenom + Nom | Identifiant affiche |
| `password_hash` | string | BCrypt | Mot de passe hashe (pour les tests d'auth) |
| `is_premium` | boolean | 20% true | Statut abonnement premium |
| `created_at` | datetime | 12 derniers mois | Date d'inscription |
| `last_activity` | datetime | 0-30 jours | Derniere connexion |
| `engagement_score` | float | 0-100 | Score d'engagement (freq. connexion, actions) |
| `satisfaction_score` | int | 1-10 | Score satisfaction simule |
| `objective` | string | {WEIGHT_LOSS, MUSCLE_GAIN, MAINTENANCE, IMPROVE_SLEEP} | Objectif personnel |

### Rationale de generation

- **20% premium** : ratio realiste pour un modele freemium sante
- **Distribution d'age** : gaussienne centree sur 30 ans (sigma 10), [18-70]
- **Engagement** : correle positivement avec le statut premium (les premium s'engagent plus)
- **Last activity** : 70% actifs dans les 7 derniers jours, 20% entre 7 et 30 jours, 10% inactifs

### Justification

Les datasets Kaggle ne fournissent pas de donnees d'engagement, de conversion premium ou de satisfaction. Ces metriques sont pourtant necessaires pour les **KPIs business** exiges par le sujet (engagement, conversion premium, satisfaction). Un generateur faker permet de produire des donnees realistes et controlees.

---

## Matrice de couverture

| Exigence du sujet | Source(s) | Couvert |
|---|---|---|
| Base nutritionnelle (aliments, apports, macronutriments) | Source 1 (Daily Food) | OUI |
| Besoins dietetiques, recommandations | Source 2 (Diet Reco) | OUI |
| Catalogue exercices (type, intensite, equipement) | Source 3 (ExerciseDB) | OUI |
| Profils utilisateurs (age, sexe, poids, taille) | Sources 2 + 4 + 6 | OUI |
| Objectifs personnalises (perte de poids, prise de masse...) | Source 6 (faker) | OUI |
| Donnees d'activite (steps, calories, duree) | Sources 4 + 5 | OUI |
| Donnees biometriques (BPM, fat%, BMI) | Source 4 (Gym Members) | OUI |
| KPIs business (engagement, conversion, satisfaction) | Source 6 (faker) | OUI |
| Au moins 2 sources de donnees | 5 sources externes + 1 generee | OUI |
| Formats heterogenes | CSV + JSON imbrique + genere | OUI |

---

## Diagramme de flux global

```
┌──────────────────────────────────────────────────────────────────────────┐
│                              EXTRACTION                                  │
│                                                                          │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐   │
│  │ Daily Food   │ │ Diet Reco    │ │ Gym Members  │ │ Fitness      │   │
│  │ CSV          │ │ CSV          │ │ CSV          │ │ Tracker CSV  │   │
│  └──────┬───────┘ └──────┬───────┘ └──────┬───────┘ └──────┬───────┘   │
│         │                │                │                │            │
│  ┌──────┴────────────────┴────────────────┴────────────────┴──────┐     │
│  │                     pandas.read_csv()                          │     │
│  └───────────────────────────┬────────────────────────────────────┘     │
│                              │                                          │
│  ┌──────────────┐            │           ┌──────────────┐              │
│  │ ExerciseDB   │            │           │ Faker        │              │
│  │ JSON         │            │           │ Generator    │              │
│  └──────┬───────┘            │           └──────┬───────┘              │
│         │ json.load()        │                  │ script python        │
│         │                    │                  │                       │
└─────────┼────────────────────┼──────────────────┼──────────────────────┘
          │                    │                  │
          v                    v                  v
┌──────────────────────────────────────────────────────────────────────────┐
│                           NETTOYAGE (Pandas)                             │
│                                                                          │
│  • Validation structure (colonnes, types)                                │
│  • Suppression doublons                                                  │
│  • Bornes metier (calories, BPM, poids, taille)                         │
│  • Normalisation (unites, formats dates, genre)                          │
│  • Conversion (Height m -> cm, arrays JSON -> texte)                     │
│  • Fusion sources 4+5 (deduplication inter-sources)                      │
│  • Croisement sources 2+4 (rapprochement profils)                        │
│                                                                          │
└─────────────────────────────────┬────────────────────────────────────────┘
                                  │
                                  v
┌──────────────────────────────────────────────────────────────────────────┐
│                        CHARGEMENT (PostgreSQL)                            │
│                                                                          │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐   │
│  │   users      │ │  nutrition   │ │  exercises   │ │  biometric   │   │
│  │              │ │  _entries    │ │  (referentiel│ │  _entries    │   │
│  └──────────────┘ └──────────────┘ │  + _entries) │ └──────────────┘   │
│                                     └──────────────┘                    │
│  ┌──────────────┐                                                       │
│  │  etl_logs    │  <- Metriques du pipeline (lignes lues/inserees/      │
│  │              │     rejetees, duree, statut)                           │
│  └──────────────┘                                                       │
└──────────────────────────────────────────────────────────────────────────┘
```
