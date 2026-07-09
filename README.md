# DevOps-Quiz-CI-Lib

Reusable CI/CD pipeline library for the
[DevOps-Quiz](https://github.com/Slavk11/DevOps-Quiz) platform.

Every service in the platform — 9 Go microservices and 3 frontend apps —
runs the **same versioned pipeline** included from this repository instead of
maintaining its own copy-pasted `.gitlab-ci.yml`. A service's own CI file
shrinks to a few lines:

```yaml
# .gitlab-ci.yml of any platform service
include:
  - project: 'devopstrain-project/ci-lib'
    ref: v1              # pinned library version
    file: 'service.yml'

variables:
  SERVICE_NAME: quiz
```

Fixing a pipeline bug or adding a security scan here updates **every service
at once** — no chasing 12 repositories.

## Pipeline

```
build → test → package → migrate → deploy → (review env | prod)
```

| Stage | What happens |
|---|---|
| **build** | Compile the service, cache dependencies between runs |
| **test** | Unit / integration tests |
| **package** | Build the Docker image, tag with commit SHA, push to the registry |
| **migrate** | Apply DB migrations (backward-compatible by contract) |
| **deploy** | `helm upgrade --install` from [Charts](https://github.com/Slavk11/DevOps-Quiz-Charts) with the service's values file |

### Per-branch behaviour

- **feature branch** → deploys to a **dynamic review environment**
  (`<branch>.devops-quiz.com`), created on first push
- **merge** → review environment is destroyed, pipeline deploys to `dev`
- **release/tag** → deployment to `prod`; the `show` service goes through the
  [canary flow](https://github.com/Slavk11/DevOps-Quiz-Charts#canary-releases)
  instead of a direct rollout

## Structure

```
.
├── service.yml           # full pipeline for a standard platform service
├── templates/
│   ├── build.yml         # build & cache
│   ├── test.yml
│   ├── package.yml       # docker build/push
│   ├── migrate.yml       # DB migrations
│   ├── deploy.yml        # helm deploy, one job per environment type
│   └── review.yml        # dynamic environments: create / destroy
└── README.md
```

Templates are composable: a service with no database simply doesn't include
the migrate template; frontend apps swap the Go build for their own
(React build for `app`, Hugo for `land`).

## Runners

All jobs run on **self-hosted runners inside the cluster**
([DevOps-Quiz-Gitlab-Runner](https://github.com/Slavk11/DevOps-Quiz-Gitlab-Runner)):
build caches live close to the jobs, and deploy jobs reach the cluster
without exposing it to the outside world.

## Design decisions

- **Library over copy-paste** — pipeline logic exists exactly once;
  services declare *what* they are, the library decides *how* to ship them
- **Pinned versions (`ref:`)** — services consume a tagged library version,
  so a library change never silently breaks someone's pipeline; upgrades are
  explicit
- **Migrations in the pipeline, not in app startup** — a deploy fails fast on
  a bad migration instead of crash-looping pods; combined with
  backward-compatible migrations this makes `helm rollback` always safe
- **Review envs from the same templates** — a review environment is a normal
  deploy with different values, not a separate hand-maintained script

---

Part of the **[DevOps-Quiz](https://github.com/Slavk11/DevOps-Quiz)** platform ·
[Terraform](https://github.com/Slavk11/DevOps-Quiz-Terraform) ·
[Infra](https://github.com/Slavk11/DevOps-Quiz-Infra) ·
[Charts](https://github.com/Slavk11/DevOps-Quiz-Charts) ·
[GitLab Runner](https://github.com/Slavk11/DevOps-Quiz-Gitlab-Runner) ·
[Frontend](https://github.com/Slavk11/DevOps-Quiz-Frontend)
