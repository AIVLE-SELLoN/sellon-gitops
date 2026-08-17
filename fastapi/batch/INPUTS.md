# FastAPI batch workload contracts and input gate

This directory owns the two batch CronJobs only (`classification-worker`,
`daily-batch`). It is independently renderable — `kubectl kustomize
fastapi/batch` does not depend on `fastapi/core`, `apps/`, `core/policies`,
or the top-level kustomization — and, like `fastapi/core`, is deliberately
not wired into `apps/fastapi.yaml` yet.

No cluster Seed Job was added here. If a one-time ChromaDB/data seed step is
ever needed, it is a separate, explicit decision — not implied by adding
these CronJobs.

## Critical labeling contract

Every CronJob's `spec.jobTemplate.spec.template.metadata.labels` must
include `app=fastapi-ai-node`. The platform's ChromaDB NetworkPolicy
(owned outside this repo) allows port 8000 by that label; a Pod template
missing it is not rejected — it just times out reaching ChromaDB, with no
error to point at the label. Both CronJobs in this directory hardcode the
label directly in the Pod template (in addition to the `labels` kustomize
transformer) so this can't silently regress if the transformer's field-spec
coverage for CronJob ever changes.

## Open items — do not resolve arbitrarily

| Required input | Confirmed value | Owner |
| --- | --- | --- |
| Docker Hub account/namespace for `sellon-ai-node` | TBD — placeholder `<DOCKERHUB_NAMESPACE>` in both `image:` fields, same as `fastapi/core` | Backend/Infra |
| `daily-batch` cron schedule (time-of-day) | TBD — placeholder `<CRON_SCHEDULE_TBD>` | AI team / Backend |
| `daily-batch` process entrypoint | TBD — placeholder `<DAILY_BATCH_ENTRYPOINT_TBD>`; no access to the AI source repo from this GitOps repo | AI team |

Do not set the `command` fields to a guessed module path, and do not set the
`schedule` fields to a guessed cron expression — a wrong module path only
surfaces at runtime as `CrashLoopBackOff`; this is the same reasoning
already applied to the Web/Consumer Deployments in `fastapi/core`.

## Classification Worker — not deployment-ready

`00-classification-worker-cronjob.yaml` now has confirmed
`schedule: "0 2 * * *"` with `timeZone: Asia/Seoul` (02:00 KST),
`concurrencyPolicy: Forbid`, `activeDeadlineSeconds: 1800`, and
`command: ["python", "scripts/classification_worker.py"]`. MQ/Chroma/S3
env/config are deliberately not wired into this workload (explicit
instruction) — this supersedes the earlier speculative note above about
needing `CHROMA_HOST`/`CHROMA_PORT`.

Despite those confirmed fields, this manifest must not be treated as
deployable until every item below is closed:

| Blocker | Status | Owner |
| --- | --- | --- |
| Docker Hub account/namespace | TBD — placeholder `<DOCKERHUB_NAMESPACE>` | Backend/Infra |
| `scripts/` Dockerfile PR merge status | **Unverified from this session.** This GitOps repo/session has no access to the AI source repo's PR system, so merge status could not be checked. | AI team |
| linux/amd64 production image actually pushed to Docker Hub | **Unverified from this session.** No registry access from here to confirm an image exists for the merged PR's SHA, or that it was built for linux/amd64. | AI team / Backend |
| Raw PostgreSQL DSN / account Secret | TBD — see `fastapi/core/INPUTS.md` "Raw DB input gate" (env var name(s), Secret name, Secret key(s) all unconfirmed). Placeholders `<RAW_DB_DSN_ENV_VAR_TBD>` / `<RAW_DB_USERNAME_ENV_VAR_TBD>` / `<RAW_DB_PASSWORD_ENV_VAR_TBD>` / `<RAW_DB_SECRET_NAME_TBD>` / their `*_SECRET_KEY_TBD` counterparts are scaffolding for both a DSN-shaped and a split-credential-shaped contract; drop whichever doesn't apply once confirmed — do not guess the shape now. | AI team |
| AI code Postgres support | TBD — if the AI code is still SQLite-only (per `fastapi/core/INPUTS.md`, `RAW_DB_PATH` is currently a SQLite-only setting, not an operating contract), this CronJob must stay inactive even after the DSN/creds above are filled in. | AI team |
| `requests`/`limits` (250m/256Mi requests, 500m/512Mi limits) | Estimates only, not measured against real workload behavior. Revisit once the worker has run and actual CPU/memory usage is known. | Backend |

## Confirmed by existing contract

`daily-batch`'s `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` env vars are
wired to the `fastapi-s3-credentials` Secret (declared in
`fastapi/core/11-s3-external-secret.yaml`, referenced here by name only —
not owned by this directory), per the "Daily batch (future workload)" row
already recorded in `fastapi/core/INPUTS.md`. `dockerhub-pull-secret` is
referenced the same way, by name only, per the same project-wide contract
Web and Consumer already use.
