# ADR-002 : Dashboard React fait maison (pas Metabase/Superset)

- **Statut :** Accepte
- **Date :** 2026-03-16
- **Decideurs :** Equipe HealthAI Coach (4 developpeurs)

## Contexte

Le projet exige :
1. Un **tableau de bord interactif** affichant des KPIs (utilisateurs, nutrition, fitness, business)
2. Une **interface d'administration** pour visualiser les flux de donnees, corriger les anomalies, et exporter en CSV/JSON
3. Le **respect strict des normes d'accessibilite RGAA niveau AA**, explicitement mentionne comme critere determinant de l'evaluation

La question est : faut-il utiliser un outil de dashboarding existant (Metabase, Apache Superset, Grafana) ou construire le dashboard directement dans le frontend React ?

## Decision

**Nous construisons le dashboard directement dans React avec la librairie Recharts**, integre a l'interface d'administration dans une application React unique.

## Alternatives evaluees

### Alternative A : Metabase (embarque ou iframe)

| Critere | Evaluation |
|---|---|
| Rapidite de mise en place | Excellente — connexion directe a PostgreSQL, dashboards en mode clic |
| Qualite des visualisations | Bonne — graphiques interactifs natifs |
| Personnalisation | Limitee — themes et layouts contraints par Metabase |
| Accessibilite RGAA AA | **Non conforme** — Metabase genere son propre HTML/JS non controle |
| Integration a l'interface admin | Difficile — necessiterait un iframe ou une seconde application |
| Empreinte Docker | Lourde — Metabase necessite sa propre JVM + base H2/PostgreSQL interne |

**Raison du rejet :** L'impossibilite de garantir la conformite RGAA AA est redhibitoire. Metabase genere du HTML que nous ne controlons pas : contrastes des graphiques, navigation clavier, attributs ARIA, alternatives textuelles — tout est hors de notre portee. Le sujet indique explicitement que la conformite RGAA AA est un "critere determinant" de l'evaluation.

### Alternative B : Apache Superset

| Critere | Evaluation |
|---|---|
| Rapidite de mise en place | Moyenne — configuration plus complexe que Metabase |
| Qualite des visualisations | Excellente — grande variete de graphiques |
| Personnalisation | Moyenne — CSS personnalisables mais le DOM reste genere par Superset |
| Accessibilite RGAA AA | **Non conforme** — meme problematique que Metabase |
| Integration | Complexe — application web independante avec son propre systeme d'auth |
| Empreinte Docker | Tres lourde — Redis + Celery + PostgreSQL interne + serveur web |

**Raison du rejet :** Memes raisons d'accessibilite que Metabase, avec en plus une empreinte infrastructure beaucoup plus lourde. Le deploiement Docker Compose deviendrait significativement plus complexe, risquant de depasser la contrainte des 30 minutes.

### Alternative C : Grafana

| Critere | Evaluation |
|---|---|
| Rapidite de mise en place | Bonne — connexion PostgreSQL native |
| Qualite des visualisations | Excellente pour le monitoring, moins adaptee aux KPIs business |
| Accessibilite RGAA AA | **Non conforme** |
| Orientation | Monitoring/observabilite — pas concue pour des dashboards metier |

**Raison du rejet :** Accessibilite non conforme. Grafana est concue pour du monitoring technique/infrastructure, pas pour des dashboards metier avec KPIs business.

## Consequences

### Positives

- **Controle total sur l'accessibilite** — Chaque element HTML est ecrit par l'equipe : `aria-label` sur les graphiques, `<table>` alternatives pour les donnees visuelles, contrastes maitrises, navigation clavier complete. La conformite RGAA AA est garantissable.
- **Application unifiee** — L'admin et le dashboard sont dans la meme application React, avec le meme systeme d'authentification JWT, la meme navigation, le meme design system. Pas de rupture d'experience utilisateur.
- **Empreinte Docker legere** — Un seul conteneur nginx sert le build React statique. Pas de JVM Metabase supplementaire ni de service Redis/Celery Superset.
- **Personnalisation illimitee** — Les graphiques, layouts et interactions sont exactement ceux dont le projet a besoin, ni plus ni moins.

### Negatives

- **Plus de code a ecrire** — Les graphiques doivent etre codes individuellement avec Recharts au lieu d'etre configures en mode clic. Cout estime : 4-5h supplementaires pour Dev 4.
- **Moins de variete graphique** — Recharts couvre les graphiques courants (line, bar, pie, area, radar) mais n'offre pas la meme richesse qu'un outil dedie (cartes geographiques, heatmaps complexes). Suffisant pour les besoins identifies.
- **Pas de mode exploration** — Avec Metabase, un admin peut creer de nouveaux dashboards ad-hoc. Notre solution ne propose que les vues predefinies. Acceptable pour le scope du projet.

### Mitigation du cout de developpement

Pour rester dans le budget horaire :
1. Utiliser des composants Recharts prets a l'emploi (`<BarChart>`, `<PieChart>`, `<LineChart>`) sans customisation excessive
2. Creer un composant `<KpiCard>` reutilisable pour les metriques simples (chiffre + tendance)
3. Prioriser : les 4 onglets KPI de base d'abord, les graphiques secondaires ensuite
4. Les donnees sont exposees par l'API en format pret a l'affichage (agregations cote Spring Boot, pas cote React)

## Criteres RGAA AA couverts par cette decision

| Critere | Comment notre approche le garantit |
|---|---|
| 1.1.1 Alternatives textuelles | `aria-label` et `<desc>` sur chaque `<svg>` Recharts |
| 1.3.1 Information et relations | Structure semantique `<main>`, `<section>`, `<h2>` pour chaque onglet |
| 1.4.3 Contraste minimum | Palette de couleurs choisie avec rapport >= 4.5:1 |
| 2.1.1 Navigation clavier | Focus gere nativement par React, tabindex sur les elements interactifs |
| 4.1.2 Nom, role, valeur | Composants React avec props ARIA correctes |

## References

- [RGAA 4.1 — Referentiel general d'amelioration de l'accessibilite](https://accessibilite.numerique.gouv.fr/methode/criteres-et-tests/)
- [Recharts Accessibility](https://recharts.org/en-US/api)
- [WCAG 2.1 AA — W3C](https://www.w3.org/TR/WCAG21/)
- [Metabase Accessibility Issues (GitHub)](https://github.com/metabase/metabase/issues?q=accessibility)
