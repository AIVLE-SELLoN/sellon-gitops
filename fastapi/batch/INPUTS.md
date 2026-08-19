# FastAPI batch workload contracts and input gate

This directory owns the three batch CronJobs (`classification-worker`,
`daily-batch`, `monthly-report`). It is independently renderable — `kubectl kustomize
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
error to point at the label. All CronJobs in this directory hardcode the
label directly in the Pod template (in addition to the `labels` kustomize
transformer) so this can't silently regress if the transformer's field-spec
coverage for CronJob ever changes.

## Confirmed image input

| Required input | Confirmed value | Owner |
| --- | --- | --- |
| Docker Hub account/namespace for `sellon-ai-node` | Confirmed: `y0njunch0i` (`image: y0njunch0i/sellon-ai-node:main-6e7b1b1`) in both fields, same as `fastapi/core` | Backend/Infra |

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
ConfigMap). `RAW_DB_PATH` (the old SQLite setting) is intentionally not
set — the AI team has confirmed the raw DB access is migrating to
PostgreSQL (see "AI code Postgres support" below), so no SQLite path is an
operating contract here.

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

Remaining inputs that are still unresolved:

| Required input | Confirmed value | Owner |
| --- | --- | --- |
| Docker Hub account/namespace | Confirmed: `y0njunch0i` (`image: y0njunch0i/sellon-ai-node:main-6e7b1b1`) | Backend/Infra |
| Raw PostgreSQL connection | Confirmed atomics: `RAW_DB_HOST`/`PORT`/`NAME`/`SSLMODE` from `fastapi-ai-node-raw-db-config`, plus `RAW_DB_USERNAME`/`PASSWORD` from `fastapi-raw-db-credentials`. `RAW_DB_DSN` is deprecated. | Infra |
| `MQ_COMPANY_ID` value | Confirmed: `1`. The Company entity PK is `@GeneratedValue(IDENTITY) Long id`; under the single-company assumption the first row is `id=1`. Verify with `SELECT` after the actual row is created. | Backend |
| `S3_COMPANY_ID` value | Confirmed: `1`. The Company entity PK is `@GeneratedValue(IDENTITY) Long id`; under the single-company assumption the first row is `id=1`. Verify with `SELECT` after the actual row is created. | Backend |
| `S3_BUCKET_NAME` value | Confirmed: `sellon-reports-dev-337658133748-ap-northeast-2-an` | Infra |
| Report bucket + per-prefix Lifecycle ownership | Unresolved. Whether the `reports/` bucket and its two Lifecycle rules (`reports/monthly-report/`, `reports/cs-guideline/`) become Terraform-managed or stay a manually created bucket has not been decided. Same class of gap as the S3 IAM user/policy ownership item already open for `fastapi-s3-credentials`. | Infra |
| `gp3` StorageClass availability | Confirmed available in-cluster | Infra |

`MQ_COMPANY_ID` and `S3_COMPANY_ID` are set to `1`: the Company entity PK is
`@GeneratedValue(IDENTITY) Long id`, and the single-company assumption makes
the first row `id=1`. Verify this with `SELECT` after the actual row is
created. Do not set raw DB values to guessed names.

## Monthly Report PDF Batch

`03-monthly-report-cronjob.yaml` runs
`scripts/generate_monthly_reports.py --stage all` at 00:00 KST on the first
day of each month, with an 08:00 KST completion target. It uses raw DB,
LLM, S3, and RabbitMQ callback contracts; it deliberately has no ChromaDB
environment variables. `--stage all` writes `data/monthly_inputs/{YYYY-MM}.json`
and reads it again in the same Pod before uploading the in-memory PDF directly
to S3, so it has no PVC.

The CronJob is `suspend: true` during the demo, so its automatic schedule does
not run even though its manifest is present. Run it manually from the CronJob
template instead; enable the schedule only after the operating team explicitly
approves it:

```bash
# Default path: REPORT_MONTH is empty, so the container calculates the prior KST month.
kubectl -n apps create job monthly-report-manual --from=cronjob/fastapi-ai-node-monthly-report
```

`--from` copies the PodTemplate. To run a specific month, first generate that
Job YAML, add the following entry to the `monthly-report` container's `env`,
then create the edited Job according to the approved deployment procedure:

```yaml
- name: REPORT_MONTH
  value: "2026-07"
```

```bash
kubectl -n apps create job monthly-report-2026-07 \
  --from=cronjob/fastapi-ai-node-monthly-report \
  --dry-run=client -o yaml > monthly-report-2026-07.yaml
```

`REPORT_MONTH` is the direct `--month` override: the container passes it as
`--month "$M"` after the Pod starts. It must use `YYYY-MM`; do not try to
append `--month` to `kubectl create job --from`, because that command only
copies the existing PodTemplate and does not alter its container arguments.

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
| Docker Hub account/namespace | Confirmed: `y0njunch0i` (`image: y0njunch0i/sellon-ai-node:main-6e7b1b1`) | Backend/Infra |
| `scripts/` Dockerfile PR merge status | **Unverified from this session.** This GitOps repo/session has no access to the AI source repo's PR system, so merge status could not be checked. | AI team |
| linux/amd64 production image actually pushed to Docker Hub | **Unverified from this session.** No registry access from here to confirm an image exists for the merged PR's SHA, or that it was built for linux/amd64. | AI team / Backend |
| Raw PostgreSQL connection env vars | Confirmed — split 5-var contract, raw-db-credentials Secret (same as `daily-batch`; see that row above). | AI team |
| AI code Postgres support | **Resolved — AI team confirmed the raw DB access is migrating from SQLite to PostgreSQL.** This supersedes the earlier finding (recorded from a prior read of `app/core/raw_db.py`/`app/batch/daily.py`, which at that time used `sqlite3.connect(...)` with no PostgreSQL path). This is a team-relayed confirmation, not independently re-verified against updated AI repo code from this session — the remaining rows below (image namespace, Dockerfile PR merge, image push) are still unverified and still block activation on their own. | AI team |
| `requests`/`limits` (250m/256Mi requests, 500m/512Mi limits) | Estimates only, not measured against real workload behavior. Revisit once the worker has run and actual CPU/memory usage is known. | Backend |

## PR 6 blockers

These two items remain unresolved as of this render/verification pass and
are recorded here as explicit blockers for PR 6 (this batch PR), not just
background open items:

| Blocker | Status |
| --- | --- |
| AI code's raw PostgreSQL connection + env contract | **Resolved.** Env contract confirmed — split 5-var contract, `raw-db-credentials` Secret (see per-workload rows above). AI team also confirmed the AI code itself is migrating off SQLite to PostgreSQL (see "AI code Postgres support" row above). |
| Worker-inclusive image (the `scripts/`-adding Dockerfile PR merge + a linux/amd64 image actually pushed) | Unresolved — unverifiable from this GitOps repo/session (no access to the AI repo's PR system or the Docker Hub registry). See "Classification Worker — not deployment-ready" above. |

`01-daily-batch-cronjob.yaml`'s raw DB blocker is closed. Both CronJobs
remain non-deployable for the separate reasons already tracked above
(Docker Hub namespace placeholder for both; unmerged/unverified worker
image for `classification-worker`) — both stay `suspend: true` until those
are closed.

## Verification findings (this pass)

- `successfulJobsHistoryLimit` / `failedJobsHistoryLimit` / `backoffLimit`
  are not set on either CronJob — both run on Kubernetes' built-in defaults
  (`successfulJobsHistoryLimit: 3`, `failedJobsHistoryLimit: 1`,
  `backoffLimit: 6` for the underlying Job). No explicit value was ever
  requested for these, so none was invented; flagged here as an open
  decision rather than left silently implicit.
- Resolved: `00-classification-worker-cronjob.yaml` and
  `01-daily-batch-cronjob.yaml` previously disagreed on the raw DB
  placeholder shape (3-way scaffold vs. single DSN). Both now use the same
  confirmed split 5-var contract (`RAW_DB_HOST`/`RAW_DB_PORT`/`RAW_DB_NAME`
  literals + `RAW_DB_USERNAME`/`RAW_DB_PASSWORD` from `raw-db-credentials`),
  so the two workloads agree.
- Resolved: the classification worker's separate SQLite-only blocker (see
  "AI code Postgres support" above) is closed — the AI team confirmed the
  raw DB access is migrating to PostgreSQL. `classification-worker` remains
  `suspend: true` regardless, for the unrelated image blockers still open
  above (Docker Hub namespace, Dockerfile PR merge, image push).

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
