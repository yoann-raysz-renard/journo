# Teetsh — feature survey

Reference notes on [Teetsh](https://teetsh.com/), the main existing SaaS covering the same
ground as journo: an all-in-one web platform for French *professeurs des écoles*
(maternelle → CM2) to prepare and run their class. Captured 2026-08-21 from the vendor's
own site plus one independent review; treat prices and module boundaries as a snapshot.

It is here as a **functional reference** — what a real product in this space covers — not as
a spec to implement. journo's subject is Kubernetes deployment; the app only needs enough
surface to be worth deploying.

## Modules

| Module | What it does |
| --- | --- |
| **Emploi du temps** | Timetable built by drag-and-drop of sessions onto slots. Custom colours and tags, automatic per-subject hour totals computed live, session duplication, multi-level and multi-class support. PDF export. |
| **Cahier journal** | The daily log. Sessions are generated automatically from the timetable, then filled in with pedagogical content; official curriculum domains and objectives are pre-loaded. Sessions copy across days and weeks. Readable remotely, so a substitute or part-time colleague can pick it up. PDF export. |
| **Fiches de préparation** | Lesson-preparation sheets in a simple text editor, with official curriculum objectives pre-registered for selection. PDF export with font choice. |
| **Progressions et programmations** | Curriculum laid out by subject, period and week across all cycles. Period boundaries are computed per school-holiday zone; content groups by domain or subject. |
| **Carnet de notes** | Assessment and progress tracking with automatic computation. Selectable or custom notation scales (percentage, out of 10, A/PA/NA). Results grouped by subject or domain, with graphs positioning each pupil against the class average. Per-pupil or whole-class PDF. |
| **Livrets LSU** | Official report booklets generated from the grade book, no re-entry. Compatible with LSU export (the national *Livret Scolaire Unique*). |
| **Carnets de réussites** | Competency/achievement booklets, the maternelle-oriented counterpart to marks. |
| **Gestion des élèves** | Rosters and customisable lists, *registre d'appel* (attendance), *trombinoscope* (class photos), *pyramide des âges*. |
| **Planner 108h** | Consolidates the statutory 108 hours — parent meetings, training, appointments — with an appointment-booking flow for families. |
| **Budget de classe** | Class budget tracking. |
| **Journal de classe (Belgium)** | Belgian variant carrying that country's competency framework. |
| **Assistant IA** | Paid add-on: an AI agent that checks work against the official programmes, drafts educational content, and writes per-pupil comments. |

## Cross-cutting capabilities

- **Collaboration** — multi-user access on one class, aimed at job-sharing pairs (*binômes*).
- **Multi-level classes** — first-class support for mixed-grade rooms, not a workaround.
- **Multi-school** — a separate journal per class across different schools.
- **Workshop rotations** — organising guided and autonomous activity groups.
- **PDF everywhere** — every artefact (journal, prep sheets, timetable, grade reports) prints.
- **Web-only, cloud-hosted** — no dedicated mobile app or offline mode is advertised.
- **RGPD** — compliance claimed, with on-demand account deletion.

## Commercial shape

| Plan | Price | Adds |
| --- | --- | --- |
| **Gratuit** | €0 | Cahier journal, emploi du temps, listes d'élèves, carnets de réussites, carnet de notes, livret & export LSU |
| **Essentiel** | €4.92/mo — €59/yr (−20%) | Progressions/programmations, fiches de préparation |
| **Intégral** | €7.42/mo — €89/yr (−20%) | Registre d'appel, planner 108h, prise de rendez-vous, budget de classe, pyramide des âges, trombinoscope, basic AI allowance |
| **Assistant IA** | +€9/mo | 6× the AI allowance included in Intégral |
| **École** | €53.10/yr per teacher (−10% from 4 licences) | Centralised licence management — activate, suspend, transfer — plus a head-teacher account and pupil distribution across classes |

A generous free tier carries the daily-driver features (journal, timetable, grades, LSU);
planning and the administrative extras are what you pay for.

## Observations relevant to journo

- The **timetable is the spine**. Sessions flow from it into the journal, and prep sheets and
  progressions hang off those sessions. Any credible data model starts there.
- **The official curriculum is reference data**, shipped with the product — domains,
  objectives, cycles, holiday zones. That is a static dataset to load, which makes it a
  natural fit for an init container, a seeded volume, or a ConfigMap in the deployment
  experiments.
- **PDF generation is a distinct workload** — CPU-spiky, latency-tolerant, easy to separate
  from the request path. A good excuse for a second Deployment, a queue, and an HPA.
- **Term/period logic depends on the holiday zone**, so there is real domain logic beyond CRUD.
- Sharing is narrow: colleagues and substitutes, not pupils. Authentication and tenancy stay
  simple — one teacher, one or more classes.

## Sources

- [teetsh.com](https://teetsh.com/) — product pages, features and capabilities
- [teetsh.com/pricing](https://teetsh.com/pricing/) — plans and per-plan feature limits
- [outilstice.com review (2021)](https://outilstice.com/2021/09/teetsh-creer-cahier-journal-gerer-sa-classe-en-ligne/) — independent walkthrough of how each tool behaves
