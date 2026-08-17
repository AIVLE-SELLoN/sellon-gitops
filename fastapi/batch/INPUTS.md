# FastAPI batch workload contracts and input gate

This directory owns the two batch CronJobs only (`classification-worker`,
`daily-batch`). It is independently renderable — `kubectl kustomize
fastapi/batch` does not depend on `fastapi/core`, `apps/`, `core/policies`,
or the top-level kustomization — and, like `fastapi/core`, is deliberately
not wired into `apps/fastapi.yaml` yet.

No cluster Seed Job was added here. If a one-time ChromaDB/data seed step is
ever needed, it is a separate, explicit decision — not implied by adding
these CronJobs.

### Dev-machine seeding procedure (no cluster Seed Job)

Because there is no cluster Seed Job in this repo, any one-time seeding of
the datastore(s) these workloads depend on (e.g. ChromaDB) must be done
from a developer's own machine via `kubectl port-forward`, not by adding a
one-off Job/CronJob here:

1. `kubectl -n default port-forward svc/chromadb 8000:8000` (ChromaDB runs
   in `default`, not `apps` — see the namespace-boundary note already
   recorded in `fastapi/core/INPUTS.md`). Adjust the service name/port for
   whatever datastore is actually being seeded.
2. Run the AI repo's seeding script locally against the forwarded port
   (e.g. `localhost:8000`), using developer-scoped credentials — never the
   CronJob's Secret-backed production credentials, and never as a cluster
   Job.
3. **Never pass `--reset`** during this procedure. No cluster Seed Job
   exists to gate or review a reset, so a developer running one locally
   against a port-forwarded connection would be resetting the live
   dataset directly. Do not script, alias, or automate `--reset` into any
   part of this procedure or CI.

This procedure is operational guidance recorded per explicit instruction in
this session; it is not derived from reading the AI repo's seeding script,
since this GitOps repo has no access to it.

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

Do not set the `command` fields to a guessed module path, and do not set the
`schedule` fields to a guessed cron expression — a wrong module path only
surfaces at runtime as `CrashLoopBackOff`; this is the same reasoning
already applied to the Web/Consumer Deployments in `fastapi/core`.

## Daily Batch — open items

`01-daily-batch-cronjob.yaml` now has confirmed `schedule: "30 2 * * *"`
with `timeZone: Asia/Seoul` (02:30 KST), `concurrencyPolicy: Forbid`,
`jobTemplate.spec.activeDeadlineSeconds: 3600`, and
`command: ["python", "-m", "app.batch.daily"]`. `--window-end` is
deliberately not part of that command — it is a manual-reprocessing flag
only, not passed on the regular schedule (see file header comment).

MQ/LLM/Chroma/S3 env now match the AI-team-confirmed names (mirroring the
literal values already established in `fastapi/core`'s ConfigMaps, since
this independently-renderable base does not envFrom another workload's
ConfigMap). `RAW_DB_PATH` (SQLite) is intentionally not set — not an
operating contract, per the "Raw DB input gate" below.

`02-daily-batch-pvc.yaml` adds a 1Gi RWO PVC (`storageClassName: gp3`)
mounted at `/app/data/batch_state`, assuming a `gp3` StorageClass already
exists in-cluster (Terraform/EBS CSI — this repo does not create
StorageClasses).

Remaining inputs, unresolved on purpose:

| Required input | Confirmed value | Owner |
| --- | --- | --- |
| Docker Hub account/namespace | TBD — placeholder `<DOCKERHUB_NAMESPACE>` | Backend/Infra |
| Raw PostgreSQL DSN env var name / Secret name / Secret key | Shape confirmed as a single DSN (not split username/password) by the AI team, but the exact names are not — placeholders `<RAW_DB_DSN_ENV_VAR_TBD>` / `<RAW_DB_SECRET_NAME_TBD>` / `<RAW_DB_DSN_SECRET_KEY_TBD>` in one place, superseding the classification worker's 3-way scaffold for this workload | AI team |
| `MQ_COMPANY_ID` value | TBD — placeholder `<MQ_COMPANY_ID_TBD>`; same "must not publish while blank" constraint recorded in `fastapi/core/INPUTS.md` "Raw DB input gate" | Backend |
| `S3_COMPANY_ID` value | TBD — placeholder `<S3_COMPANY_ID_TBD>`; same per-company gating concern, scoped to this workload's S3 upload path | Backend |
| `gp3` StorageClass availability | Assumed to exist in-cluster; not verified from this repo/session | Infra |

Do not fill `MQ_COMPANY_ID`/`S3_COMPANY_ID` with a guessed company
identifier, and do not set the raw DB placeholders to a guessed name — this
CronJob must not be treated as deployable until these, and the Docker Hub
namespace, are confirmed.

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

## PR 6 blockers

These two items remain unresolved as of this render/verification pass and
are recorded here as explicit blockers for PR 6 (this batch PR), not just
background open items:

| Blocker | Status |
| --- | --- |
| AI code's raw PostgreSQL connection + env contract | Unresolved — env var name(s), Secret name, Secret key(s) not confirmed by the AI team, and it is not confirmed whether the AI code has moved past SQLite-only. See "Raw DB input gate" in `fastapi/core/INPUTS.md` and the per-workload rows above. |
| Worker-inclusive image (the `scripts/`-adding Dockerfile PR merge + a linux/amd64 image actually pushed) | Unresolved — unverifiable from this GitOps repo/session (no access to the AI repo's PR system or the Docker Hub registry). See "Classification Worker — not deployment-ready" above. |

Neither `00-classification-worker-cronjob.yaml` nor
`01-daily-batch-cronjob.yaml` should be treated as deployable while either
blocker is open.

## Verification findings (this pass)

- `successfulJobsHistoryLimit` / `failedJobsHistoryLimit` / `backoffLimit`
  are not set on either CronJob — both run on Kubernetes' built-in defaults
  (`successfulJobsHistoryLimit: 3`, `failedJobsHistoryLimit: 1`,
  `backoffLimit: 6` for the underlying Job). No explicit value was ever
  requested for these, so none was invented; flagged here as an open
  decision rather than left silently implicit.
- `00-classification-worker-cronjob.yaml` still carries the older 3-way raw
  DB placeholder scaffold (DSN + split username/password), from before the
  AI team confirmed the DSN-only shape while defining `daily-batch`'s env.
  That shape confirmation is a fact about the AI app's config schema, not
  something specific to `daily-batch`, so it plausibly applies here too —
  but this verification pass did not change `00-classification-worker-cronjob.yaml`
  (out of scope for a verify-only pass); flagging the inconsistency for a
  follow-up edit rather than resolving it unasked.

## Confirmed by existing contract

`daily-batch`'s `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` env vars are
wired to the `fastapi-s3-credentials` Secret (declared in
`fastapi/core/11-s3-external-secret.yaml`, referenced here by name only —
not owned by this directory), per the "Daily batch (future workload)" row
already recorded in `fastapi/core/INPUTS.md`. `dockerhub-pull-secret` is
referenced the same way, by name only, per the same project-wide contract
Web and Consumer already use.
