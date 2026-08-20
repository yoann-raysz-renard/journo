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
| Application | The journo app itself (source, container image). |
| Deployment approaches | One directory per approach — plain manifests, Kustomize, Helm, GitOps, an operator, … each deploying the same app. |
| Local cluster | A throwaway Kubernetes cluster (kind / minikube / k3d) used to try the approaches out. |

Each deployment approach is meant to be self-contained and comparable: same app, same
outcome, different mechanics and trade-offs.

## Getting started

Nothing is set up yet — this README is the starting point. The pieces will be added as the
experiments happen, and this section will grow with the concrete prerequisites (container
runtime, `kubectl`, a local cluster tool) and setup steps.

## Layout

The repository is currently empty apart from this README. The intended shape:

```
app/        the journo application and its container image
deploy/     one subdirectory per deployment approach
docs/       notes and comparisons between approaches
```

## What runs automatically

Nothing yet — no CI, no schedules, no bots.

---

Earlier history: this repository previously held Rust and Go exercises. They were removed
when it was repurposed; they remain in the git history.
