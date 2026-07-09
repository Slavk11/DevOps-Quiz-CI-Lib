# DevOps-Quiz-CI-Lib

Reusable CI/CD pipeline library for the
[DevOps-Quiz](https://github.com/Slavk11/DevOps-Quiz) platform.

Every service in the platform — 9 Go microservices and 3 frontend apps —
runs a pipeline **included from this repository** instead of maintaining its
own copy-pasted `.gitlab-ci.yml`. A service's own CI file shrinks to a few
lines:

```yaml
# .gitlab-ci.yml of a backend service
include:
  - project: 'devopstrain-project/ci-lib'
    file:
      - 'backend/build.yml'
      - 'backend/migrate.yml'
      - 'common/deploy.yml'

variables:
  SERVICE_NAME: quiz
```

Fixing a pipeline bug or adding a step here updates **every service at
once** — no chasing 12 repositories.

## Structure

Templates are organized by **service type**, with shared logic factored out:

```
.
├── backend/
│   ├── build.yml         # Go build & test, image packaging
│   ├── migrate.yml       # DB migrations
│   └── deploy.yml        # backend-specific deploy steps
├── frontend/
│   ├── build.yml         # React (app) / Hugo (land) builds, static bundles
│   └── deploy.yml        # frontend-specific deploy steps
└── common/
    └── deploy.yml        # shared deploy logic: helm upgrade, environments
```

A service composes its pipeline from the templates matching its type:

| Service type | Includes |
|---|---|
| Go microservice with DB (`quiz`, `leads`, `users`…) | `backend/build` + `backend/migrate` + deploy |
| Go microservice without DB (`chrome`, `show`…) | `backend/build` + deploy |
| Frontend app (`app`, `land`, `widget`) | `frontend/build` + deploy |

## Pipeline

```
build & test → package image → [migrate] → deploy
```

| Stage | What happens |
|---|---|
| **build** | Compile/build the service, run tests, cache dependencies |
| **package** | Build the Docker image, tag with commit SHA, push to the registry |
| **migrate** | Apply DB migrations (backward-compatible by contract) — backend services with a database only |
| **deploy** | `helm upgrade --install` of the service's release from [Charts](https://github.com/Slavk11/DevOps-Quiz-Charts) with its values file |

### Per-branch behaviour

- **feature branch** → deploys to a **dynamic review environment**
  (`<branch>.devops-quiz.com`), created on first push
- **merge** → review environment is destroyed, pipeline deploys to `dev`
- **release/tag** → deployment to `prod`; the `show` service goes through the
  [canary flow](https://github.com/Slavk11/DevOps-Quiz-Charts#canary-releases)
  instead of a direct rollout

## Runners

All jobs run on **self-hosted runners inside the cluster**
([DevOps-Quiz-Gitlab-Runner](https://github.com/Slavk11/DevOps-Quiz-Gitlab-Runner)):
build caches live close to the jobs, and deploy jobs reach the cluster
without exposing it to the outside world.

## Design decisions

- **Templates by service type, not by service** — `backend/` and `frontend/`
  cover every current and future service; adding service #13 requires zero
  new CI code
- **Shared deploy in `common/`** — how the platform deploys is defined once;
  backend and frontend differ in how they *build*, not in how they *ship*
- **Migrations as a pipeline stage** — a deploy fails fast on a bad migration
  instead of crash-looping pods; combined with backward-compatible migrations
  this keeps `helm rollback` always safe
- **Library over copy-paste** — services declare *what* they are, the library
  decides *how* to ship them

---

Part of the **[DevOps-Quiz](https://github.com/Slavk11/DevOps-Quiz)** platform ·
[Terraform](https://github.com/Slavk11/DevOps-Quiz-Terraform) ·
[Infra](https://github.com/Slavk11/DevOps-Quiz-Infra) ·
[Charts](https://github.com/Slavk11/DevOps-Quiz-Charts) ·
[GitLab Runner](https://github.com/Slavk11/DevOps-Quiz-Gitlab-Runner) ·
[Frontend](https://github.com/Slavk11/DevOps-Quiz-Frontend)
