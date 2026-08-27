# journo

A learning repository for exploring **different ways of deploying an application on
Kubernetes**. The Kubernetes tooling is the subject; the application is just the thing
being deployed.

## The application

`journo` is an app for French *professeurs des écoles* (primary-school teachers) to help
them organize their class. It exists to give the deployment experiments something realistic
to carry — a real service, a database, and the usual concerns of a small web app.

## How the pieces fit

| Piece | Role |
| --- | --- |
| `app/api` | Spring Boot JSON API — the application proper. |
| `app/pdf` | Spring Boot PDF service, headless Chrome. Asynchronous, CPU-spiky, the autoscaling subject. |
| `web/` | Angular single-page front end. French UI via runtime translation catalogues. |
| Postgres, Redis | Durable data and shared sessions — two different stateful profiles to deploy. |
| `deploy/` | One directory per deployment approach — plain manifests, Kustomize, Helm, GitOps, an operator — each deploying the same app. |
| Local cluster | A throwaway **kind** cluster, scripted to recreate from nothing. |

Each deployment approach is meant to be self-contained and comparable: same app, same
outcome, different mechanics and trade-offs.

## Getting started

Nothing is built yet — the stack is decided but no code exists. Read
[docs/decisions.md](docs/decisions.md) first; it explains what is being built and why each
choice was made.

Expected prerequisites once there is something to run: a JDK and Maven, Node for the Angular
app, a container runtime, `kubectl`, and [kind](https://kind.sigs.k8s.io/). This section will
carry the exact versions and setup steps as they are pinned down.

## Layout

Only `docs/` exists so far. The intended shape:

```
pom.xml         Maven parent
app/api/        Spring Boot JSON API
app/pdf/        PDF service (headless Chrome)
app/shared/     DTOs, LSU model
web/            Angular front end
deploy/         one subdirectory per deployment approach
docs/           decisions, reference notes and comparisons
```

| Document | Contents |
| --- | --- |
| [docs/decisions.md](docs/decisions.md) | The stack and conventions, with the reasoning behind each choice. Start here. |
| [docs/teetsh-features.md](docs/teetsh-features.md) | Feature survey of Teetsh, the existing SaaS covering the same ground — used as a functional reference for what the app should do. |
| [docs/roadmap-free-plan.md](docs/roadmap-free-plan.md) | Todo list for building the app up to Teetsh free-plan parity, plus the parallel deployment track. |
| [docs/glossary.md](docs/glossary.md) | French domain vocabulary mapped to the English names used in code and schema. |

## What runs automatically

Nothing yet — the repository holds no code. Once the first service lands, GitHub Actions will
run the Maven build and tests plus the Angular build and lint on every push, and publish
container images to GHCR tagged by commit SHA. See [decisions.md](docs/decisions.md) §19.

---

Earlier history: this repository previously held Rust and Go exercises. It was reset to a
single root commit when repurposed, so that history is gone.
