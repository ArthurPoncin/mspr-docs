# Architecture Decision Records (ADR)

Ce dossier contient les decisions architecturales du projet HealthAI Coach, documentees au format ADR.

## Qu'est-ce qu'un ADR ?

Un ADR (Architecture Decision Record) capture une decision architecturale importante, son contexte, les alternatives evaluees, et les consequences attendues. Cela permet a l'equipe de comprendre **pourquoi** un choix a ete fait, pas seulement **quel** choix a ete fait.

## Index des ADR

| # | Titre | Statut | Date |
|---|---|---|---|
| [ADR-001](001-java-spring-boot-api.md) | Java Spring Boot pour l'API REST | Accepte | 2026-03-16 |
| [ADR-002](002-dashboard-react-maison.md) | Dashboard React fait maison (pas Metabase/Superset) | Accepte | 2026-03-16 |
| [ADR-003](003-orchestration-cron-simple.md) | Orchestration ETL par cron simple (pas Airflow) | Accepte | 2026-03-16 |
| [ADR-004](004-communication-via-postgresql.md) | Communication ETL-API via PostgreSQL uniquement | Accepte | 2026-03-16 |

## Statuts possibles

- **Propose** — En cours de discussion
- **Accepte** — Valide par l'equipe
- **Remplace** — Annule par un autre ADR
- **Deprecie** — Plus applicable
