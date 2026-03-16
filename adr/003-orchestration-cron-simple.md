# ADR-003 : Orchestration ETL par cron simple (pas Airflow)

- **Statut :** Accepte
- **Date :** 2026-03-16
- **Decideurs :** Equipe HealthAI Coach (4 developpeurs)

## Contexte

Le pipeline ETL doit etre automatise et journalise (exigence du sujet). La question porte sur le choix de l'outil d'orchestration : un orchestrateur complet comme Apache Airflow, ou un mecanisme plus simple (cron, `@Scheduled` Spring, ou script Python avec `schedule`).

Le sujet mentionne explicitement : "cron, Airflow, ou equivalent".

Nos pipelines consistent a :
- Importer 2-3 fichiers CSV/JSON depuis un dossier local
- Nettoyer les donnees avec Pandas
- Charger en base PostgreSQL
- Logger le resultat dans la table `etl_logs`

Il n'y a pas de dependances complexes entre pipelines, pas de branches conditionnelles, et pas de connexions a des APIs externes en temps reel.

## Decision

**Nous utilisons un cron simple dans le conteneur Docker ETL**, combine a un endpoint Spring Boot `POST /api/admin/etl/run` pour le declenchement manuel.

Concretement :
- Un `crontab` dans le conteneur Python execute `python main.py` a intervalle regulier (configurable)
- Le script Python ecrit ses logs dans la table `etl_logs` en base
- L'interface admin consulte `etl_logs` pour afficher le monitoring
- L'admin peut aussi declencher un import a la demande via l'API

## Alternatives evaluees

### Alternative A : Apache Airflow

| Critere | Evaluation |
|---|---|
| Puissance | Excellente — DAGs, retries, backfill, monitoring UI natif |
| Complexite d'installation | Elevee — webserver, scheduler, worker, metadata DB (PostgreSQL ou SQLite), Redis/Celery pour CeleryExecutor |
| Empreinte Docker | Tres elevee — minimum 3 conteneurs supplementaires |
| Courbe d'apprentissage | Significative — concepts DAG, tasks, XCom, Connections, Variables |
| Pertinence pour notre scope | Surdimensionne — 2-3 pipelines lineaires sans branchement |
| Temps de mise en place | 4-6h (installation, configuration, ecriture des DAGs, debug) |

**Raison du rejet :** Le ratio cout/benefice est inacceptable pour notre scope. Airflow est concu pour orchestrer des dizaines/centaines de pipelines avec des dependances complexes. Nous avons 2-3 scripts lineaires. L'installation et la configuration consommeraient 4-6h sur un budget total de 19h, sans apport fonctionnel reel.

### Alternative B : `@Scheduled` Spring Boot

| Critere | Evaluation |
|---|---|
| Simplicite | Excellente — une annotation Java |
| Integration | Native — dans le meme conteneur que l'API |
| Probleme | L'ETL est en Python, pas en Java. Le `@Scheduled` devrait executer un `ProcessBuilder` pour lancer le script Python, ce qui couple inutilement les deux services. |

**Raison du rejet partiel :** Utilisable pour le declenchement via l'API (`POST /api/admin/etl/run` appelle le script Python), mais pas comme orchestrateur principal. Le cron dans le conteneur ETL est plus propre architecturalement car il respecte la separation des responsabilites.

### Alternative C : Python `schedule` library

| Critere | Evaluation |
|---|---|
| Simplicite | Bonne — `schedule.every(1).day.at("02:00").do(run_pipeline)` |
| Probleme | Necessite un processus Python long (daemon) qui tourne en permanence, ce qui est moins robuste qu'un cron systeme pour un conteneur Docker. |

**Raison du rejet :** Un daemon Python est moins resilient qu'un cron systeme : si le processus Python crash, la planification s'arrete. Avec cron, chaque execution est independante.

## Consequences

### Positives

- **Zero overhead** — Pas de service supplementaire a deployer, configurer ou maintenir.
- **Fiabilite** — Cron est un outil eprouve depuis des decennies, natif dans les conteneurs Linux.
- **Simplicite Docker** — Le conteneur ETL est un simple `python:3.12-slim` avec un crontab. Build en quelques secondes.
- **Budget horaire preserve** — Aucune heure consommee sur l'installation/configuration d'un orchestrateur. Le temps economise est reinvesti dans la qualite du nettoyage des donnees et l'interface admin.
- **Facile a expliquer au jury** — Un cron est comprehensible immediatement. Airflow necesiterait d'expliquer des concepts (DAG, scheduler) qui ne sont pas le coeur du sujet.

### Negatives

- **Pas de UI de monitoring native** — Airflow offre un webserver pour visualiser l'etat des DAGs. Nous compensons avec notre propre monitoring via la table `etl_logs` et l'onglet "Monitoring ETL" dans l'interface admin React.
- **Pas de retry automatique** — Si un pipeline echoue, cron ne retente pas automatiquement. Mitigation : le script Python gere ses propres try/except et log le statut FAILED. L'admin peut relancer manuellement via l'interface.
- **Pas de backfill** — Pas de mecanisme natif pour rejouer un import sur une plage de dates passees. Acceptable pour notre scope (imports de fichiers statiques, pas de flux temps reel).

### Perspectives d'evolution

Si HealthAI Coach devait passer a l'echelle (dizaines de sources, pipelines complexes avec dependances), la migration vers Airflow serait justifiee. Le code ETL actuel (extractors, cleaners, loaders) est structure de maniere modulaire et pourrait etre encapsule dans des tasks Airflow sans reecriture majeure.

## Implementation

```dockerfile
# etl/Dockerfile
FROM python:3.12-slim

RUN apt-get update && apt-get install -y cron && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Crontab : execution quotidienne a 2h du matin
COPY crontab /etc/cron.d/etl-cron
RUN chmod 0644 /etc/cron.d/etl-cron && crontab /etc/cron.d/etl-cron

CMD ["cron", "-f"]
```

```
# etl/crontab
0 2 * * * cd /app && python main.py >> /var/log/etl.log 2>&1
```

## References

- [Docker + Cron best practices](https://docs.docker.com/samples/)
- [Apache Airflow Architecture](https://airflow.apache.org/docs/apache-airflow/stable/concepts/overview.html)
