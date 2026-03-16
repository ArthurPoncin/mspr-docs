# ADR-005 : Utilisation des 5 sources de donnees fournies (pas seulement 2)

- **Statut :** Accepte
- **Date :** 2026-03-16
- **Decideurs :** Equipe HealthAI Coach (4 developpeurs)

## Contexte

Le cahier des charges fournit 5 sources de donnees et exige d'en utiliser **au moins 2**, avec une justification du choix. Les sources fournies sont :

| # | Source | Format | Volume |
|---|---|---|---|
| 1 | Daily Food & Nutrition Dataset | CSV | ~1 000 entrees |
| 2 | Diet Recommendations Dataset | CSV | 1 000 profils |
| 3 | ExerciseDB API Repository | JSON (imbrique) | 11 000+ exercices |
| 4 | Gym Members Exercise Dataset | CSV | 973 echantillons |
| 5 | Fitness Tracker Dataset | CSV | ~1 000 entrees |

La question est : combien de sources utiliser et lesquelles ?

## Decision

**Utiliser les 5 sources fournies**, completees par un generateur de donnees simulees (faker) pour les metriques business.

### Mapping retenu

| Source | Table(s) cible | Axe couvert |
|---|---|---|
| Daily Food & Nutrition | `nutrition_entries` | Nutrition |
| Diet Recommendations | `users` (enrichissement) + `biometric_entries` | Nutrition + Biometrie |
| ExerciseDB API | `exercises` (table referentiel) | Sport |
| Gym Members Exercise | `users` + `biometric_entries` + `exercise_entries` | Sport + Biometrie |
| Fitness Tracker | `exercise_entries` (fusion avec source 4) | Sport |
| Donnees simulees (faker) | `users` (complement business) | Business |

## Alternatives evaluees

### Alternative A : 2 sources minimum (Daily Food + Gym Members)

| Critere | Evaluation |
|---|---|
| Conformite | Respecte le minimum exige |
| Couverture | Nutrition + Sport + Biometrie couverts |
| Pipeline ETL | Simple : 2 CSV plats |
| Volume | ~2 000 lignes |
| Valeur pedagogique | Faible — pas de cas de nettoyage complexe |

**Raison du rejet :** Respecter le strict minimum ne demontre pas d'ambition. Le pipeline ETL serait trivial (2 `read_csv` + quelques filtres). Pas de cas de fusion, pas de JSON imbrique, pas de croisement de sources. Le jury risque de juger le travail data insuffisant.

### Alternative B : 3 sources (Daily Food + Gym Members + ExerciseDB)

| Critere | Evaluation |
|---|---|
| Conformite | Au-dela du minimum |
| Couverture | Bonne |
| Pipeline ETL | Plus interessant : CSV + JSON imbrique |
| Volume | ~12 000+ lignes |
| Valeur pedagogique | Correcte — aplatissement JSON |

**Raison du rejet :** Laisse de cote Diet Recommendations (profils sante, pathologies, recommandations IA) et Fitness Tracker (steps, distance). On perd le cas de fusion inter-sources (sources 4+5) qui est pedagogiquement riche.

### Alternative C : 5 sources (retenue)

| Critere | Evaluation |
|---|---|
| Conformite | Largement au-dela du minimum |
| Couverture | Complete sur les 3 axes |
| Pipeline ETL | Riche : CSV + JSON, fusion, croisement, deduplication |
| Volume | ~14 000+ lignes |
| Valeur pedagogique | Elevee — 4 cas de nettoyage non triviaux |

## Consequences

### Positives

- **Couverture complete des 3 axes** — Nutrition (sources 1+2), Sport (sources 3+4+5), Biometrie (sources 2+4). Aucun axe n'est couvert par une seule source, ce qui renforce la robustesse.

- **4 cas de nettoyage remarquables a presenter au jury :**
  1. **Fusion de datasets quasi-identiques** (sources 4+5) — deduplication inter-sources, normalisation de colonnes aux noms legerement differents
  2. **Aplatissement de JSON imbrique** (source 3) — arrays de strings (`targetMuscles`, `equipments`) transformes en colonnes texte ou relations
  3. **Croisement de sources heterogenes** (sources 2+4) — rapprochement de profils par criteres demographiques
  4. **Generation de donnees complementaires** (faker) — enrichissement avec des metriques business absentes des datasets

- **Diversite de formats** — 4 CSV + 1 JSON imbrique. Le pipeline ETL traite des extracteurs differents (`csv_extractor`, `json_extractor`), ce qui demontre sa genericite.

- **Volume suffisant** — ~14 000+ enregistrements au total. Assez pour des KPIs significatifs, pas trop pour depasser les capacites de PostgreSQL en local.

- **Demontrer de l'ambition** avec le minimum de surcout. Les 5 sources sont fournies et documentees : l'effort supplementaire par rapport a 2 sources est principalement dans les cleaners specifiques (5 fichiers au lieu de 2), ce qui reste gerable.

### Negatives

- **Plus de cleaners a ecrire** — 5 extracteurs/cleaners au lieu de 2, soit environ 3-4h de dev supplementaire. Mitigation : les cleaners CSV partagent une structure commune (validation bornes, deduplication), seules les regles metier changent.

- **ExerciseDB necessite un fork GitHub + extraction API** — La donnee n'est pas un simple fichier a telecharger. Mitigation : extraire une fois, sauvegarder en `exercises.json` local, le pipeline lit le fichier local ensuite.

- **Risque de decouvrir des problemes de qualite sur une source** — Un dataset pourrait etre inutilisable (colonnes manquantes, donnees trop sales). Mitigation : telecharger et auditer les 5 sources en priorite (premiere heure du projet). Si une source est problematique, on la retire sans impacter les autres.

- **Fusion sources 4+5 potentiellement complexe** — Si les colonnes ne matchent pas exactement, le mapping sera manuel. Mitigation : la fusion est un bonus pedagogique. En cas de difficulte, on garde les deux sources separees avec un champ `source` pour les distinguer.

## Priorisation en cas de manque de temps

Si le temps manque, les sources sont priorisees dans cet ordre :

| Priorite | Source | Raison |
|---|---|---|
| **P0 (obligatoire)** | Daily Food & Nutrition | Coeur du domaine nutrition |
| **P0 (obligatoire)** | Gym Members Exercise | Couvre sport + biometrie + profils |
| **P1 (importante)** | Donnees simulees (faker) | Necessaire pour les KPIs business |
| **P1 (importante)** | ExerciseDB | Montre le traitement JSON, table referentiel |
| **P2 (bonus)** | Diet Recommendations | Enrichissement profils sante |
| **P2 (bonus)** | Fitness Tracker | Fusion inter-sources (pedagogique) |

Avec P0 seul, on respecte le minimum de 2 sources et on couvre les 3 axes. Chaque niveau supplementaire ajoute de la valeur sans casser ce qui existe.

## References

- Cahier des charges MSPR TPRE501, section "Jeux de donnees de reference"
- Document detaille : [`data/SOURCES.md`](../data/SOURCES.md)
