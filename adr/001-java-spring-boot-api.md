# ADR-001 : Java Spring Boot pour l'API REST

- **Statut :** Accepte
- **Date :** 2026-03-16
- **Decideurs :** Equipe HealthAI Coach (4 developpeurs)

## Contexte

Le projet HealthAI Coach necessite un backend API REST pour :
- Exposer les donnees nettoyees (nutrition, sport, biometrie) en CRUD
- Servir des endpoints d'agregation pour le tableau de bord (KPIs)
- Gerer l'authentification et les autorisations (roles USER / ADMIN)
- Fournir des endpoints d'administration (validation donnees, export, declenchement ETL)
- Generer automatiquement une documentation OpenAPI (Swagger), livrable obligatoire

Le pipeline ETL est deja en Python (Pandas etant incontournable pour la manipulation de donnees). Le choix du langage et framework pour l'API est donc une decision independante.

Le sujet est rattache au Bloc E6.1 (Data Science, RNCP36581) mais certains membres de l'equipe suivent egalement le cursus CDA (Concepteur Developpeur d'Applications) et doivent demontrer des competences en developpement d'applications structurees.

## Decision

**Nous utilisons Java 21 avec Spring Boot 3** pour le backend API REST.

Modules Spring retenus :
- **Spring Web** — controleurs REST
- **Spring Data JPA** (Hibernate) — acces donnees et ORM
- **Spring Security** — authentification JWT et autorisations par role
- **Spring Validation** — validation des DTOs entrants (`@Valid`)
- **Springdoc OpenAPI 2** — generation automatique de la documentation Swagger UI
- **Flyway** — migrations de schema SQL versionnees

## Alternatives evaluees

### Alternative A : Python FastAPI

| Critere | Evaluation |
|---|---|
| Coherence avec l'ETL Python | Excellent — un seul langage pour tout le backend |
| Documentation OpenAPI | Excellente — generation native, zero configuration |
| Performance | Bonne (async natif via uvicorn) |
| Maitrise equipe | Moyenne — l'equipe a plus d'experience en Java |
| Ecosysteme securite | Correct (dependances tierces pour JWT, pas de framework unifie) |
| Structure projet | Libre — pas de convention imposee, risque de code spaghetti sous contrainte de temps |

**Raison du rejet :** L'equipe a une maitrise significativement plus forte de Java et Spring Boot. Avec seulement 19h de budget, le risque d'apprentissage sur FastAPI (decorateurs, Pydantic, Depends, async patterns) aurait impacte la productivite. La coherence de langage est un avantage theorique, mais la maitrise reelle prime en contexte contraint.

### Alternative B : Node.js Express

| Critere | Evaluation |
|---|---|
| Coherence avec le frontend React | Bon — meme langage JS/TS |
| Documentation OpenAPI | Moyenne (swagger-jsdoc avec annotations manuelles) |
| Performance | Bonne |
| Maitrise equipe | Faible |
| Ecosysteme securite | Fragmente (passport.js, chaque strategie a installer separement) |
| Typage | Faible sans TypeScript, et TypeScript ajoute un cout de configuration |

**Raison du rejet :** Maitrise insuffisante par l'equipe. Ecosysteme securite fragmente. La generation OpenAPI demande plus de configuration manuelle que Springdoc.

### Alternative C : Python Django REST Framework

| Critere | Evaluation |
|---|---|
| Coherence avec l'ETL Python | Excellent |
| Documentation OpenAPI | Bonne (drf-spectacular) |
| Performance | Correcte |
| Maitrise equipe | Faible sur Django specifiquement |
| Structure projet | Tres structuree (apps, models, serializers, views) |
| ORM | Django ORM — puissant mais syntaxe differente de JPA |

**Raison du rejet :** Courbe d'apprentissage Django specifique (apps, migrations Django, serializers) sans apport fonctionnel justifiant l'investissement en temps. L'equipe n'a pas d'experience Django.

## Consequences

### Positives

- **Productivite maximale** — L'equipe code dans un langage maitrise, reduisant les blocages et le debugging lie a la meconnaissance du framework.
- **Robustesse structurelle** — Spring Boot impose un cadre (injection de dependances, annotations, couches service/controller/repository) qui limite les erreurs d'architecture sous pression de temps.
- **Typage fort a la compilation** — Les erreurs de type sont detectees avant l'execution, ce qui est particulierement pertinent pour un contexte de donnees de sante ou la fiabilite du traitement compte.
- **Generation OpenAPI automatique** — Springdoc genere la documentation Swagger a partir des annotations `@RestController` sans effort supplementaire. Livrable obligatoire couvert nativement.
- **Securite integree** — Spring Security offre un cadre complet (filtres, providers, annotations `@PreAuthorize`) au lieu d'assembler des librairies tierces.
- **Migrations SQL explicites** — Flyway s'integre nativement a Spring Boot : les migrations SQL sont executees automatiquement au demarrage de l'application.
- **Valorisation professionnelle** — Spring Boot est le standard des SI d'entreprise en France. Demontrer sa maitrise dans un contexte data renforce le profil CDA et DIADS.

### Negatives

- **Deux langages dans le projet** — Le pipeline ETL est en Python, l'API en Java. Cela implique deux ecosystemes a maintenir, deux Dockerfiles, et une separation nette des responsabilites dans l'equipe.
- **Verbosity Java** — Java est plus verbeux que Python ou TypeScript (DTOs, getters/setters, annotations). Mitigation : utiliser Lombok ou les records Java 21 pour reduire le boilerplate.
- **Temps de build** — La compilation Java + le build Maven/Gradle est plus lent qu'un demarrage Python. Mitigation : profil Spring Boot DevTools pour le hot-reload en dev.
- **Potentielle question du jury** — Le choix Java sera probablement questionne dans un contexte Data Science. L'equipe doit preparer une reponse argumentee (voir section "Arguments pour le jury" ci-dessous).

### Neutres

- **Communication ETL <> API** — Les deux services ne communiquent pas directement entre eux. Ils partagent uniquement la base PostgreSQL. Le choix de langage de l'API n'impacte donc pas le pipeline ETL. Ce decouplage est documente dans [ADR-004](004-communication-via-postgresql.md).

## Arguments prepares pour le jury

> **Q : Pourquoi Java et pas Python pour l'API, sachant que votre ETL est deja en Python ?**

Reponse structuree :

1. **Pragmatisme** — Nous avons 19h par personne. Notre maitrise de Spring Boot nous permet de livrer un backend robuste dans ce delai. Apprendre FastAPI aurait consomme du temps au detriment des fonctionnalites.

2. **Separation des responsabilites** — L'ETL et l'API ont des responsabilites distinctes. L'ETL manipule des données brutes (Pandas excelle pour cela). L'API expose des donnees structurees et gere l'authentification (Spring excelle pour cela). Utiliser le bon outil pour chaque job est un choix d'ingenierie, pas une incoherence.

3. **Typage et fiabilite** — Dans un contexte de donnees de sante, nous voulons un maximum de verifications a la compilation. Le typage fort de Java + les validations Spring (`@Valid`, `@NotNull`, `@Min`) offrent une securite supplementaire par rapport a un langage dynamique.

4. **Standard industriel** — En entreprise, les SI de sante en France utilisent majoritairement des stacks Java. Ce choix est coherent avec la realite du marche.

## References

- [Spring Boot 3 Reference](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Springdoc OpenAPI](https://springdoc.org/)
- [Flyway with Spring Boot](https://documentation.red-gate.com/fd/quickstart-how-flyway-works-184127223.html)
- [Java 21 Record Classes](https://docs.oracle.com/en/java/javase/21/language/records.html)
