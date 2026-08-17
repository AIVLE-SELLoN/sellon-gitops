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
| `classification-worker` cron schedule | TBD — placeholder `<CRON_SCHEDULE_TBD>` | AI team / Backend |
| `daily-batch` cron schedule (time-of-day) | TBD — placeholder `<CRON_SCHEDULE_TBD>` | AI team / Backend |
| `classification-worker` process entrypoint | TBD — placeholder `<CLASSIFICATION_WORKER_ENTRYPOINT_TBD>`; no access to the AI source repo from this GitOps repo | AI team |
| `daily-batch` process entrypoint | TBD — placeholder `<DAILY_BATCH_ENTRYPOINT_TBD>`; no access to the AI source repo from this GitOps repo | AI team |
| `classification-worker` env/ConfigMap contract | TBD — no ConfigMap was added for this workload. It plausibly needs `CHROMA_HOST`/`CHROMA_PORT` (that's the reason the NetworkPolicy label matters at all), but that is inference, not a confirmed contract — nothing was invented here | AI team |

Do not set the `command` fields to a guessed module path, and do not set the
`schedule` fields to a guessed cron expression — a wrong module path only
surfaces at runtime as `CrashLoopBackOff`; this is the same reasoning
already applied to the Web/Consumer Deployments in `fastapi/core`.

## Confirmed by existing contract

`daily-batch`'s `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` env vars are
wired to the `fastapi-s3-credentials` Secret (declared in
`fastapi/core/11-s3-external-secret.yaml`, referenced here by name only —
not owned by this directory), per the "Daily batch (future workload)" row
already recorded in `fastapi/core/INPUTS.md`. `dockerhub-pull-secret` is
referenced the same way, by name only, per the same project-wide contract
Web and Consumer already use.
