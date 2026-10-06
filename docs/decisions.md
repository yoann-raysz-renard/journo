# Decisions

The stack and conventions, with the reasoning that produced them. Decided 2026-08-21.

Reading order matters: each decision constrains the ones below it. Where a choice was made
*because* it teaches something about Kubernetes rather than because it is the simplest way to
ship, that is stated — this repo optimises for the deployment lesson, not for velocity.

| # | Decision | Choice |
| --- | --- | --- |
| 1 | Language of code, schema, API, logs | English; French only in the UI |
| 2 | Runtime | Java, Spring Boot |
| 3 | Datastore | PostgreSQL |
| 4 | UI | Angular SPA against a JSON API |
| 5 | PDF generation | Separate Java service |
| 6 | PDF engine | Headless Chrome |
| 7 | Repository layout | Maven multi-module |
| 8 | Container images | Hand-written Dockerfiles |
| 9 | Sessions | Spring Session + Redis |
| 10 | Migrations | Liquibase |
| 11 | UI translation | ngx-translate runtime catalogues |
| 12 | Local cluster | kind, plain manifests first |
| 13 | Java / Spring Boot versions | Java 25 LTS, Spring Boot 4.1 |
| 14 | Base image | Temurin JRE on Debian |
| 15 | Postgres, first pass | Hand-written StatefulSet |
| 16 | Ingress controller | ingress-nginx |
| 17 | API contract | Code-first, generated Angular client |
| 18 | Testing | Unit plus Testcontainers |
| 19 | CI | Build, test, push images to GHCR |
| 20 | Redis durability | Ephemeral, no persistence |

## 1. English in code, French in the UI

Code, database schema, API payloads and log messages are English. The French vocabulary
teachers use appears only in the UI, through the translation layer. Mapping in
[glossary.md](glossary.md).

Two boundaries where French legitimately remains: reference **data** values (a subject really
is named « Questionner le monde ») and the LSU export's spec-fixed element names, which stay
confined to a mapping layer at the export boundary.

## 2. Java, Spring Boot

Chosen over Go partly for familiarity, but mainly because the JVM makes the Kubernetes
lessons real. A Go binary starts instantly in a 20 MB image, so probes, resource limits and
autoscaling latency never actually bite. With Spring Boot they do: heap sizing against
`resources.limits`, `startupProbe` versus `readinessProbe` when boot takes several seconds,
`OOMKilled` versus a heap `OutOfMemoryError`, and HPA reaction time when a fresh pod is not
useful for a while.

What it gives us directly: Actuator health groups mapping onto liveness and readiness,
graceful shutdown as a configuration property, and first-class Liquibase integration.

Same framework for both services. GraalVM native image is parked as a **later experiment** —
a like-for-like comparison on the same cluster, not a starting position.

## 3. PostgreSQL

The stateful-workload question is one of the experiments, so the datastore needs to be worth
deploying three ways: hand-rolled StatefulSet, the CloudNativePG operator, and an external
managed instance. Also the better fit for the school-period and holiday-zone date logic.

SQLite was rejected: it pins the app to one writer on one volume, caps the web tier at one
replica, and removes most of the stateful lessons.

## 4. Angular SPA against a JSON API

Angular CDK ships `DragDrop`, and the timetable — the one screen with genuinely hard
interaction — is the deciding factor. Typed forms suit the grade book too.

Accepted consequence: **three deployables plus Postgres and Redis**, an Ingress path split
between `/api/*` and `/*`, and a real API contract to keep honest. More moving parts, which
here is the point.

The usual objection to an SPA — print layout duplicated client-side — does not apply, because
decision 5 gives print layout to the PDF service entirely. Angular never renders for print.

## 5. PDF generation in a separate Java service

The app calls it over HTTP; generation is asynchronous with a job record the UI polls.

Rationale: PDF work is CPU-spiky and latency-tolerant, so it has no business competing with
web requests. Isolating it also produces the queue-plus-HPA workload that makes the
autoscaling experiments meaningful rather than theoretical. Server-side templates in this
service own all print layout.

## 6. Headless Chrome as the PDF engine

Full modern CSS — flexbox, grid, `@page` rules, web fonts — so print layout is written the
way the rest of the front end is.

Accepted cost, and it is not small: **an image around 1 GB carrying a browser**, and roughly
300–500 Mi per concurrent render. Memory limits and concurrency caps on this service will need
real tuning. That tuning is itself instructive, and it makes the HPA demonstration far more
vivid than a lightweight renderer would.

`openhtmltopdf` was the alternative — a ~200 Mi hermetic image, but CSS 2.1 only, meaning
tables and floats for every layout.

## 7. Maven multi-module

```
pom.xml            parent
  app/api/         Spring Boot API
  app/pdf/         PDF service
  app/shared/      DTOs, LSU model
web/               Angular
deploy/            one subdirectory per deployment approach
docs/
```

Shared code is a module dependency, not a published artifact, so one build produces
everything and CI stays simple while the attention goes to deployment. Maven over Gradle for
work muscle memory; the cost is verbosity and slower incremental builds.

## 8. Hand-written Dockerfiles

Multi-stage, explicit non-root user, a deliberately chosen JRE base, explicit
`-XX:MaxRAMPercentage`. Jib and buildpacks are both more ergonomic and both hide precisely
the layering and base-image decisions worth making by hand at least once. Revisit later as a
comparison.

## 9. Spring Session + Redis

Sessions live in Redis so any replica serves any request: the API scales to N replicas and
rolling updates do not log everyone out.

Consequence: **Redis is a second stateful component to deploy**, which is welcome — a
different persistence profile from Postgres, and a legitimate place to practise the
difference between cache-like and durable state.

Sticky sessions were rejected as a starting point, though the failure mode is worth
demonstrating deliberately at some stage: pin by cookie affinity, then watch a rolling update
sign everyone out.

## 10. Liquibase

Changelogs with rollback support and preconditions. Run as a **standalone command**, so it
becomes an init container or a migration `Job` rather than something the application does at
startup — the app must never migrate its own schema when N replicas start at once.

Note against the grain of decision 1: prefer `sql`-formatted changesets over XML/YAML
abstractions where practical, so the reviewable artefact stays real Postgres DDL.

## 11. ngx-translate

JSON catalogues keyed by English identifiers, loaded at runtime. One build artefact whatever
the locale, so neither the image nor the Ingress needs to know about languages. Only `fr` will
exist, but no user-facing string is ever a literal in a template.

`@angular/localize` is the official route and faster at runtime, but it wants a build and a
bundle per locale — a build matrix for a single language.

## 12. kind, plain manifests first

kind is the reference implementation, scripted to recreate from nothing, and loads local
images directly.

The first deployment approach is **hand-written manifests**: Deployment, Service, Ingress,
ConfigMap, Secret, probes, resource requests and limits, and the migration Job. Every field
written by hand once, before Kustomize, Helm or a GitOps controller generates any of them.
Sequence in [roadmap-free-plan.md](roadmap-free-plan.md) §6.

## 13. Java 25 LTS, Spring Boot 4.1

Java 25 is the current LTS, supported by Temurin until at least September 2031. Spring Boot
4.1 (June 2026) has a Java 17 baseline but tests AOT and GraalVM native images on Java 25
specifically, which keeps the native-image comparison in decision 2 open.

Java 26 was rejected as non-LTS — end of life within months and thinner base-image
availability. Java 21 plus Boot 3.5 would match most production estates today, at the cost of
a known upgrade already waiting.

## 14. Temurin JRE on Debian

`eclipse-temurin:25-jre-<debian>` for both services.

The decisive constraint is not the API but the PDF service: **Chrome and Playwright do not
officially support Alpine/musl**, so that service needs glibc regardless. Using one base
family for both keeps the Dockerfiles comparable and avoids two sets of surprises.

Distroless is the more interesting security story — no shell, minimal surface — but it cannot
host Chrome, so it would apply to at most one of the two services. Worth revisiting for the
API alone once both images exist and can be measured. `jlink` custom runtimes are smaller
still and the most educational, but `jdeps` analysis fights Spring's reflection.

## 15. Postgres as a hand-written StatefulSet, first pass

StatefulSet, headless Service, `volumeClaimTemplates`, probes and a Secret, all written by
hand. The point is to meet ordered startup, stable network identity and what a PVC actually
binds to — so that CloudNativePG later reads as a comparison rather than as magic.

No failover and no backups in this pass. That is acceptable for a dev cluster and is exactly
the gap the operator experiment should close.

## 16. ingress-nginx

kind documents it directly and it is the most widely deployed controller, so external examples
match what is actually running. Handles the `/api/*` versus `/*` split, and supports cookie
affinity — needed for the deliberate demonstration in decision 9 of why sticky sessions hurt.

Gateway API is where the ecosystem is heading and is worth a later experiment; Ingress is
still what most material and most clusters use today.

## 17. Code-first, generated Angular client

Spring controllers are the source of truth; springdoc emits the OpenAPI document; the Angular
service layer is generated from it with `ng-openapi-gen`.

The payoff is that front-end drift becomes a **compile error** rather than a runtime 404 — the
main risk the SPA split in decision 4 introduces. The cost is that the spec is a by-product,
so it is only as good as the annotations on the controllers.

Generated client code is build output: generate it into a dedicated directory and do not hand-
edit it.

## 18. Unit tests plus Testcontainers

Domain logic — school-period boundaries from holiday zones, grading scales, timetable hour totals —
tested in isolation. Everything touching persistence tested against **real Postgres and real
Redis** in containers. No H2 pretending to be Postgres.

This is what makes the Liquibase changesets genuinely tested rather than merely executed at
startup, which matters more here than usual given decision 10. Boot 4's Testcontainers support
also drives the local dev loop against the same services.

Playwright end-to-end tests are deferred, despite Playwright already being present for the PDF
service. Revisit for the drag-and-drop timetable, which is the one screen unit tests cannot
meaningfully cover.

## 19. GitHub Actions: build, test, and push images to GHCR

Maven build and tests, Angular build and lint, then container images built and pushed to GHCR
on main, tagged by commit SHA.

Pushing images is not optional cosmetics: the **GitOps approach in the roadmap requires a
registry**, because Argo CD or Flux pulls images rather than reading a local Docker daemon.
Building them from the start avoids retrofitting the pipeline at exactly the point the
deployment work gets interesting.

Deploy manifests reference images by digest where possible; SHA tags are the minimum.

## 20. Redis ephemeral, no persistence

A Deployment with `emptyDir`, `appendonly no`, and no PVC. Sessions are disposable: losing
them logs teachers out, which is irritating rather than damaging.

This is deliberate contrast, not laziness — two stateful components deployed two different
ways, side by side, so the difference between durable state and cache-like state is visible in
the manifests themselves.

## Still open

Lower stakes, and none of them block writing code:

- Java package naming and Maven coordinates.
- Angular version and component library, if any beyond the CDK.
- How the Ingress is reached locally — `nip.io`, a hosts entry, or port-forwarding — and
  whether TLS is worth doing with a local CA.
- Secret handling once GitOps arrives: plain `Secret` objects will not do when manifests live
  in a public repository. Sealed Secrets, SOPS, or External Secrets.
- Observability: whether Actuator metrics get a Prometheus stack, which the HPA work may want
  anyway for custom metrics.
