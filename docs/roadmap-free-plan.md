# Roadmap — reaching Teetsh free-plan parity

A todo list for building the journo app up to the feature set Teetsh gives away for free.
Feature reference: [teetsh-features.md](teetsh-features.md).

Headings below keep the French product names a teacher would recognise; the code name for
each is given alongside. Everything technical is in English — see the language decision in §0 and the
full mapping in [glossary.md](glossary.md).

**Target scope — the six free-plan features:** emploi du temps, cahier journal, listes
d'élèves, carnet de notes, carnets de réussites, livret & export LSU.

Remember what this repo is for: the app is the *payload* for Kubernetes deployment
experiments. Build the thinnest version that is still honestly useful — a real database, a
real background workload, real config and secrets — and resist gold-plating the product.
The deployment track at the bottom runs in parallel; you do not need feature completeness
before deploying something.

## 0. Decisions

**All settled** — see [decisions.md](decisions.md) for each choice and why:
Java/Spring Boot · PostgreSQL · Angular SPA + JSON API · separate PDF service on headless
Chrome · Maven multi-module · hand-written Dockerfiles · Spring Session + Redis · Liquibase ·
ngx-translate · kind with plain manifests first · English code, French UI only.

Also settled: Java 25 LTS · Spring Boot 4.1 · Temurin JRE on Ubuntu · Postgres as a
hand-written StatefulSet first · Gateway API with Envoy Gateway · code-first OpenAPI with a generated Angular
client · unit tests plus Testcontainers · GitHub Actions pushing images to GHCR · Redis
ephemeral · `dev.journo` packages, organised by feature.

- [x] Agree the [glossary](glossary.md) terms you will need before writing the first changeset.
- [ ] Skim the short **Still open** list at the end of `decisions.md` — Angular version, local
      Gateway hostname, GitOps secret handling, observability. None block starting.

## 1. Foundations

- [ ] Maven parent plus `app/api`, `app/pdf`, `app/shared`; `web/` for Angular.
- [ ] API service: Actuator health groups wired to liveness and readiness,
      `server.shutdown=graceful`, structured JSON logging.
- [ ] Config strictly from environment variables, no profile files baked into the image.
      Keep credentials separate from plain config so they map cleanly onto `Secret` vs.
      `ConfigMap`.
- [ ] Liquibase changelog, runnable as a **standalone command** — that becomes the init
      container or migration `Job`. The app must never migrate at startup with N replicas.
- [ ] Dockerfiles per service: multi-stage, non-root user, `eclipse-temurin:25-jre-noble`
      (Ubuntu), explicit `-XX:MaxRAMPercentage`.
- [ ] springdoc emitting the OpenAPI document, and `ng-openapi-gen` wired into the web build.
- [ ] Testcontainers for Postgres and Redis, driving both the test suite and the local dev loop.
- [ ] GitHub Actions: Maven verify, Angular build and lint, then images pushed to GHCR by SHA.
- [ ] PDF service: Chrome image size and per-render memory measured early — budget
      ~300–500 Mi per concurrent render and cap concurrency accordingly.
- [ ] Redis for sessions, wired through Spring Session.
- [ ] Angular app with ngx-translate configured before the first template is written.

## 2. Reference data

Both datasets are static inputs, which makes them a natural fit for a seed `Job`, an init
container, or a mounted `ConfigMap`.

- [ ] **School calendar.** Load holiday dates and zones (A/B/C) from the Ministry open-data
      [calendrier scolaire dataset](https://data.education.gouv.fr/explore/dataset/fr-en-calendrier-scolaire/).
      Needed to derive the school periods (*trimestres*/*semestres*) every other module reports on.
- [ ] **Curriculum.** Cycles, subjects, strands and objectives from the official *programmes*.
      Decide how deep to go — full objective trees are a large data-entry job; a subject +
      strand skeleton is enough to be credible.
- [ ] Reference **data** keeps its official French labels — a subject really is named
      « Questionner le monde ». That is content in a row, not an identifier: English column
      names, French values, and no attempt to translate the programmes.

## 3. Domain model

Build this as one vertical slice before adding breadth.

- [ ] Core entities: teacher → school → class(es) → pupils; subject/strand; timetable slot;
      lesson (*séance*); assessment; school period.
- [ ] **Multi-level classes** from the start. Retrofitting a class that spans CP *and* CE1
      into a single-level model is painful, and it is the normal case in rural schools.
- [ ] The **timetable is the spine**: lessons derive from slots, and everything else hangs
      off lessons. Model that relationship first.

## 4. The six features

Suggested order — each builds on the last.

### Emploi du temps — `timetable`
- [ ] Weekly grid: slots with subject, time range, colour, tag.
- [ ] Drag-and-drop placement and resizing.
- [ ] Live per-subject hour totals (teachers must justify statutory volumes).
- [ ] Duplicate a slot; copy a whole week.
- [ ] Multi-level and multi-class views.

### Cahier journal — `daily_log`
- [ ] Auto-generate the day's lessons from the timetable.
- [ ] Per-lesson pedagogical content: objectives, activities, materials, notes.
- [ ] Copy a lesson to another day or week.
- [ ] Day and week views; jump to any date.
- [ ] Read-only share link for a substitute — the one sharing feature the free plan needs.

### Listes d'élèves — `pupil` / `class_group`
- [ ] Pupil CRUD: name, date of birth, level, class.
- [ ] CSV import/export — nobody types 28 pupils by hand twice.
- [ ] Custom lists and groups (for workshop rotations later).

### Carnet de notes — `grade_book`
- [ ] Record assessments against a lesson or a subject/strand.
- [ ] Selectable scales: percentage, out of 10, A/PA/NA, plus a custom scale.
- [ ] Automatic aggregation by subject and by strand.
- [ ] Per-pupil view positioning them against the class average.

### Carnets de réussites — `achievement_record`
- [ ] Competency framework per cycle (this is the maternelle-facing counterpart to marks).
- [ ] Mark a competency acquired / in progress / not acquired, with a date.
- [ ] Per-pupil booklet view.

### Livret & export LSU — `periodic_report`
- [ ] Periodic assessment (*bilan périodique*) per pupil per school period, assembled from the grade
      book with no re-entry.
- [ ] Generate the LSU import XML, class by class — filename pattern `import-lsun-aaaa-mm-jj.xml`,
      carrying directors, classes, pupils, periods, domains, teachers, pathways and the
      periodic/end-of-cycle assessments.
- [ ] Validate against the official schema before export. **Read the spec first** —
      [FORMAT IMPORT LSU 1ER DEGRE](https://www.toutatice.fr/toutatice-portail-cms-nuxeo/binary/Spec_1er+degre_Editeurs_LSUN+import+bilan-V1.22.pdf?type=FILE) —
      this is a compliance format, not something to reverse-engineer from a sample.
- [ ] Treat this as the **riskiest item**: it is the only feature with an external contract
      you do not control. Consider it a stretch goal and ship the other five first.
- [ ] **The one place English stops.** LSU element and attribute names are fixed by the
      official spec and are French. Keep them confined to a mapping layer at the export
      boundary — serialisation tags or an explicit translation step — so the spec's
      vocabulary never reaches the domain model or the schema.

## 5. Export & printing

- [ ] PDF for the cahier journal, the timetable, and grade reports (per pupil and whole class).
- [ ] Print-oriented CSS as the single source of layout, rendered by the PDF service.
- [ ] Asynchronous generation with a job record the UI can poll — this is the workload that
      justifies a second Deployment and autoscaling.

## 6. Deployment track (the actual point)

Run this alongside the features; do not wait for parity.

- [ ] Local **kind** cluster, scripted to recreate from nothing, with Envoy Gateway installed.
- [ ] Image build and load into the cluster; pin by digest where possible.
- [ ] **Approach 1** — plain manifests: Deployments for api/pdf/web, Services, a Gateway and
      HTTPRoutes splitting `/api/*` from `/*`, ConfigMap, Secret, probes (including a `startupProbe` —
      Spring Boot will need it), resource requests/limits, and the Liquibase migration `Job`.
- [ ] Postgres as a hand-written StatefulSet with `volumeClaimTemplates`; Redis as a plain
      Deployment on `emptyDir`. Two stateful components, two deliberately different shapes.
- [ ] **Approach 2** — Kustomize: a base plus dev/prod overlays.
- [ ] **Approach 3** — Helm chart with values for the same overlays.
- [ ] **Approach 4** — GitOps with Argo CD or Flux reconciling from this repo.
- [ ] Postgres the other two ways: the CloudNativePG operator, then an external managed instance.
- [ ] HPA on the PDF worker, driven by a deliberately generated backlog.
- [ ] Write up each approach's trade-offs in `docs/` — that comparison is the deliverable.

## Explicitly out of scope

Paid-plan features. Do not drift into them: progressions et programmations, fiches de
préparation, registre d'appel, planner 108h, appointment booking, budget de classe, pyramide
des âges, trombinoscope, and anything AI.
