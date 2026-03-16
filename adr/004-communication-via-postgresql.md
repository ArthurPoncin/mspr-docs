# ADR-004 : Communication ETL-API via PostgreSQL uniquement

- **Statut :** Accepte
- **Date :** 2026-03-16
- **Decideurs :** Equipe HealthAI Coach (4 developpeurs)

## Contexte

Le projet comporte deux composants backend dans deux langages differents :
- **ETL** en Python (Pandas) — importe, nettoie et charge les donnees
- **API** en Java (Spring Boot) — expose les donnees et gere l'authentification

Ces deux composants doivent echanger des donnees. La question est : comment communiquent-ils ?

## Decision

**L'ETL et l'API ne communiquent pas directement entre eux.** Leur seul point de contact est la base de donnees PostgreSQL partagee.

```
ETL Python ──────> PostgreSQL <────── API Spring Boot
  (ecrit)           (source de         (lit)
                     verite)
```

- L'ETL ecrit dans les tables metier (`nutrition_entries`, `exercise_entries`, `biometric_entries`) et dans la table de monitoring (`etl_logs`)
- L'API lit ces memes tables pour les exposer via REST
- Il n'y a **aucun appel HTTP** entre l'ETL et l'API
- Il n'y a **aucune file de messages** (RabbitMQ, Kafka) entre les deux

L'unique exception : l'endpoint `POST /api/admin/etl/run` de l'API peut declencher un import en executant le script Python via un mecanisme systeme (appel Docker exec ou commande interne). Ce flux est initie par l'utilisateur admin, pas par un processus automatique.

## Alternatives evaluees

### Alternative A : L'ETL appelle l'API via HTTP

```
ETL Python ──HTTP POST──> API Spring Boot ──> PostgreSQL
```

| Critere | Evaluation |
|---|---|
| Couplage | Fort — l'ETL doit connaitre les endpoints, les DTOs, et le format d'authentification de l'API |
| Validation | Double — Pandas valide a l'extraction, Spring valide a la reception |
| Performance | Inferieure — serialisation JSON + reseau + deserialisation pour chaque ligne |
| Complexite | L'ETL doit gerer un token JWT pour s'authentifier aupres de l'API |

**Raison du rejet :** Ajoute du couplage, de la complexite et de la latence sans benefice reel. L'ETL insere des milliers de lignes ; passer par l'API HTTP pour chaque insertion est un anti-pattern pour du chargement batch.

### Alternative B : File de messages (RabbitMQ / Kafka)

```
ETL Python ──publish──> RabbitMQ ──consume──> API Spring Boot ──> PostgreSQL
```

| Critere | Evaluation |
|---|---|
| Decouplage | Excellent — les deux services sont independants |
| Complexite infrastructure | Elevee — un service supplementaire a deployer et maintenir |
| Pertinence | Surdimensionne — justifie pour du streaming temps reel, pas pour des imports batch de fichiers statiques |

**Raison du rejet :** Overhead d'infrastructure disproportionne pour le scope du projet. Pas de besoin de traitement en temps reel.

## Consequences

### Positives

- **Decouplage maximal** — L'ETL et l'API sont totalement independants. On peut modifier, redemarrer ou supprimer l'un sans impacter l'autre.
- **Simplicite** — Pas de protocole de communication a definir, pas d'authentication inter-services, pas de gestion d'erreurs reseau entre composants.
- **Performance** — L'ETL ecrit directement en base via `psycopg2` / `SQLAlchemy` avec des insertions batch (`COPY` ou `executemany`). Aucune serialisation intermediaire.
- **Testabilite** — Chaque composant peut etre teste independamment. L'ETL est testable avec une base PostgreSQL de test. L'API egalement.
- **Schema comme contrat** — Le schema PostgreSQL (gere par Flyway) est le contrat d'interface entre les deux services. Si le schema change, les migrations Flyway le propagent, et les deux cotes doivent s'adapter.

### Negatives

- **Coherence du schema** — L'ETL Python doit connaitre la structure exacte des tables (noms de colonnes, types). Si Flyway modifie le schema, le code ETL doit etre mis a jour manuellement. Mitigation : les colonnes et types sont definis dans un fichier `config.py` centralise cote ETL.
- **Pas de validation API a l'insertion ETL** — Les donnees inserees par l'ETL ne passent pas par les validations `@Valid` de Spring. L'ETL doit implementer ses propres validations (regles metier dans les cleaners). Mitigation : les regles metier sont documentees et appliquees dans les deux sens.
- **Pas de notification temps reel** — L'API ne sait pas immediatement quand un import ETL est termine. Elle lit `etl_logs` a la demande. Acceptable pour un dashboard qui n'a pas besoin de rafraichissement en temps reel sub-seconde.

### Garanties de coherence

Pour eviter les problemes de coherence entre l'ETL et l'API :

1. **Flyway est la source de verite** pour le schema. Les migrations sont versionnees dans le repository.
2. **Le champ `status`** (`BRUT`, `NETTOYE`, `VALIDE`, `REJETE`) garantit que l'API ne sert que des donnees validees. Meme si l'ETL insere des donnees en cours de nettoyage, le dashboard ne les affiche qu'apres validation admin.
3. **Les contraintes SQL** (NOT NULL, CHECK, FOREIGN KEY) sont la derniere ligne de defense contre les donnees invalides.

## References

- [Shared Database pattern — Microservices.io](https://microservices.io/patterns/data/shared-database.html)
- [PostgreSQL COPY for batch inserts](https://www.postgresql.org/docs/current/sql-copy.html)
