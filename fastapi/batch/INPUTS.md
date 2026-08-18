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

`S3_ENABLED: "true"` is set explicitly and must not be dropped. The AI
repo's `app/reporting/s3_uploader.py` defaults it to `false`, and
`ensure_s3_ready()` checks it before anything else in the PDF pipeline,
raising `S3NotConfiguredError` when it is off — so omitting this key makes
every Daily run fail at the upload step. `S3_REGION` is pinned to
`ap-northeast-2` for the same class of reason: the presigned URL is signed
with this region while the bucket host is built from it, and a mismatch
surfaces as a 403 on the link while the upload itself still succeeds.

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
| `S3_COMPANY_ID` value | TBD — placeholder `<S3_COMPANY_ID_TBD>`. Env var name confirmed against the AI repo (`app/reporting/s3_uploader.py` reads `S3_COMPANY_ID`); `ensure_s3_ready()` raises `S3NotConfiguredError` on a blank value rather than uploading to a guessed path, so leaving the placeholder in place fails safe. | Backend |
| `S3_BUCKET_NAME` value | TBD — placeholder `<S3_BUCKET_NAME_TBD>`. The report bucket is **not declared in the INFRA Terraform repo** (only the Terraform state bucket in `bootstrap/` is), and the Notion S3 documents define the folder layout and per-prefix Lifecycle retention (monthly-report 6 months, cs-guideline 7 days) without naming the bucket. The AI code carries an account-ID-bearing dev default; do not fall back to it. | Infra |
| Report bucket + per-prefix Lifecycle ownership | Unresolved. Whether the `reports/` bucket and its two Lifecycle rules (`reports/monthly-report/`, `reports/cs-guideline/`) become Terraform-managed or stay a manually created bucket has not been decided. Same class of gap as the S3 IAM user/policy ownership item already open for `fastapi-s3-credentials`. | Infra |
| `gp3` StorageClass availability | Confirmed available in-cluster | Infra |

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
| AI code Postgres support | **Confirmed: still SQLite-only.** The AI repo's `app/core/raw_db.py` opens the raw DB with `sqlite3.connect(f"{path.as_uri()}?mode=ro", uri=True)` against `settings.raw_db_path`, and `app/batch/daily.py` imports `sqlite3` directly. There is no PostgreSQL connection path in the code at all, so this CronJob must stay inactive regardless of what DSN env names are later agreed. | AI team |
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
  DB placeholder scaffold (DSN + split username/password), while
  `01-daily-batch-cronjob.yaml` uses a single DSN placeholder. The two
  should agree, since the shape is a property of the AI app's config schema
  rather than of either workload.

  However, the "single DSN confirmed by the AI team" claim recorded against
  `daily-batch` has **no supporting evidence in the AI code**: there is no
  PostgreSQL connection path there to have a DSN shape (see the
  "AI code Postgres support" row above). Reconcile the two placeholders only
  after re-confirming the shape with the AI team — do not standardize on the
  DSN form purely to remove the inconsistency.

## Confirmed by existing contract

`daily-batch`'s `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` env vars are
wired to the `fastapi-s3-credentials` Secret (declared in
`fastapi/core/11-s3-external-secret.yaml`, referenced here by name only —
not owned by this directory), per the "Daily batch (future workload)" row
already recorded in `fastapi/core/INPUTS.md`. `dockerhub-pull-secret` is
referenced the same way, by name only, per the same project-wide contract
Web and Consumer already use.

## ServiceAccount dependency (cross-PR)

Both CronJobs set `serviceAccountName: fastapi-ai-node` under
`spec.jobTemplate.spec.template.spec`. That ServiceAccount is declared in
`fastapi/rbac` (PR 5), not here — referenced by name only, the same way the
Secrets above are.

`kubectl kustomize fastapi/batch` still renders standalone, but **deployment
requires PR 5 to be merged first**: a Pod naming a ServiceAccount that does
not exist is never created, so the CronJob would produce Jobs that never
start a Pod.

Omitting the line does not error — the Pod silently falls back to the
namespace's `default` ServiceAccount, which mounts an API token these
batch workloads never use, defeating the `automountServiceAccountToken:
false` guarantee the `fastapi/rbac` ServiceAccount exists to provide.
