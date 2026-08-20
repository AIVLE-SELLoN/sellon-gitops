# FastAPI integrated deployment verification

Observed on the live `sellon-eks` cluster on 2026-08-20. The per-directory
VERIFICATION.md files under `core/` and elsewhere record *rendering* checks
made before anything was applied; this file records what the cluster actually
reports once ArgoCD owns the manifests.

## ArgoCD Applications

| Application | Sync | Health |
| --- | --- | --- |
| `root` | Synced | Healthy |
| `common` | Synced | Healthy |
| `fastapi` | Synced | Healthy |
| `rabbitmq` | OutOfSync | Healthy |

`rabbitmq` is addressed under "Known deviations" below. The other three are
clean.

## Workloads

Both long-running Pods had been up 17h with zero restarts before this change,
across two different nodes:

| Pod | Ready | Restarts | Node |
| --- | --- | --- | --- |
| `fastapi-ai-node-web` | 1/1 | 0 | `ip-10-0-29-224` |
| `fastapi-ai-node-consumer` | 1/1 | 0 | `ip-10-0-42-216` |

Zero restarts matters specifically for `startupProbe`: an under-provisioned
startup budget shows up as a restart loop during cold start, and none
occurred.

Idle resource usage, for reference when the estimated CronJob
requests/limits are revised:

| Pod | CPU | Memory |
| --- | --- | --- |
| `fastapi-ai-node-web` | 2m | 164Mi |
| `fastapi-ai-node-consumer` | 1m | 106Mi |

These are idle figures from long-running services. They are not a basis for
sizing the batch workloads, whose load profile differs; measure those from an
actual run.

Three nodes are `Ready` on v1.31.14-eks-254016e.

## Scheduled workloads

| CronJob | Schedule | Timezone | Suspend | Last schedule |
| --- | --- | --- | --- | --- |
| `daily-batch` | `30 2 * * *` | Asia/Seoul | false | 8h ago |
| `classification-worker` | `0 2 * * *` | Asia/Seoul | false (this change) | — |
| `monthly-report` | `0 0 1 * *` | Asia/Seoul | true | — |

`daily-batch` is the strongest signal available here: it fired on its own
schedule and completed with no active or failed Job left behind. That single
run exercises the image, the ServiceAccount, the raw PostgreSQL contract, and
the `Asia/Seoul` timezone field together, which is why the
`classification-worker` release below is not a blind one.

`monthly-report` stays suspended by design — it is triggered manually as a
one-off Job for the demo rather than waiting for the first of the month.

## Secret delivery

All six ExternalSecrets report `SecretSynced` / `Ready=True`:

| Namespace | ExternalSecret | Store type |
| --- | --- | --- |
| `apps` | `dockerhub-pull-secret` | ClusterSecretStore |
| `apps` | `fastapi-llm-credentials` | ClusterSecretStore |
| `apps` | `fastapi-raw-db-credentials` | ClusterSecretStore |
| `apps` | `fastapi-s3-credentials` | ClusterSecretStore |
| `default` | `dockerhub-pull-secret` | ClusterSecretStore |
| `argocd` | `argocd-github-app-repository` | SecretStore |

This confirms the whole chain end to end: Secrets Manager -> IRSA -> ESO ->
Kubernetes Secret. The `argocd` entry additionally confirms the private
repository credential works, which is what lets the root Application read this
repository at all.

## Image tag

The tag now lives in one place, `fastapi/kustomization.yaml`:

```yaml
images:
  - name: y0njunch0i/sellon-ai-node
    newTag: main-9cb5baa
```

It previously appeared in five Pod templates across `core/` and `batch/`,
where a partial edit would have left workloads running mixed revisions. The
per-file references are deliberately kept so each base still renders on its
own; the override wins for anything ArgoCD applies.

Verified by rendering with a throwaway tag and confirming all five image
references changed together, then reverting.

## Scripts blocker — closed

`classification-worker` was suspended because it was unverifiable from this
repository whether the published image actually contained the script it
invokes. Checked directly against the registry:

```
docker run --rm y0njunch0i/sellon-ai-node:main-9cb5baa ls -la /app/scripts
```

`classification_worker.py` (67567 bytes) is present, along with a `prompts/`
directory and the rest of the tooling. The blocker is closed and `suspend` is
released to `false` in this change.

Note the tag: this was confirmed on `main-9cb5baa`, not on the previously
pinned `main-6e7b1b1`. If the tag is rolled back, re-confirm the script is
present in whatever image the rollback lands on rather than assuming it.

## Recorded revisions

| | Value |
| --- | --- |
| ArgoCD revision | `fd649bfac98bad1d36961357c06cf5f349812619` |
| Running digest | `sha256:8e43bc485dbaa4f1350e33e8bf1a8e3318bf2a09c86c787818b2ce4969b8199e` |
| Target digest | `sha256:d29031712b8388fad15ce48f95d44e0ac3363f6c23f1f3c01336d028e6a4fe26` |

**The runtime observations in this file predate the tag change.** The Pods
inspected here run the first digest, which is `main-6e7b1b1`; the tag in this
change points at the second, `main-9cb5baa`. Merging replaces both Pods, so
re-confirm `/health`, the consumer subscription, and readiness afterwards
rather than treating the results below as still current.

## Runtime checks

| Check | Result |
| --- | --- |
| Web `GET /health` | `200 {"status":"ok"}`, called on `localhost:8080` inside the Pod |
| Consumer RabbitMQ | `컨슈머 시작 queue=ai.inbound binding=feedback.#` |
| Consumer handlers | `feedback.recommendation.reviewed`, `feedback.report.created` |
| raw DB PostgreSQL 5432 | reached and authenticated — see below |
| `RAW_DB_DSN` residue | none as an env key; the five matches in the tree are comments recording that it is deprecated |
| NetworkPolicies | 6 present — one per workload plus the shared DNS policy |
| Daily batch PVC | `fastapi-ai-node-daily-batch-state` Bound, 1Gi, RWO, gp3 |

The consumer line carries more than it looks: reaching a named queue with a
binding means the connection, the `ai-user` credential the operator
generated, the `app` vhost, and the NetworkPolicy egress into `default` all
worked. None of those can be established from the manifests alone.

## Monthly report — blocked upstream

Run manually rather than waiting for the first of the month:

```
kubectl -n apps create job --from=cronjob/fastapi-ai-node-monthly-report <name>
```

The Job failed within seconds:

```
psycopg.errors.UndefinedTable: relation "voc_document" does not exist
  app/reporting/monthly_aggregator.py:209  list_product_groups
```

Read this as two separate results.

The raw DB contract passes. Reaching `UndefinedTable` means the connection
opened, the credentials from `fastapi-raw-db-credentials` authenticated, the
5432 egress rule allowed the traffic, and a query executed on the server. A
broken contract fails as `OperationalError` or a timeout, not as a missing
relation.

The workload itself cannot run. `monthly_aggregator` queries a
`voc_document` table the raw database does not have. This is an
application/schema mismatch with nothing for the manifests to fix — either
the table has yet to be created or the code names one that has since changed.
Raised with the AI team.

`monthly-report` therefore stays `suspend: true`, now for a substantive
reason rather than as a precaution. S3 upload and the
`publish_report_generated` event are consequently unverified: execution stops
before either is reached.

## Known deviations

**`rabbitmq` OutOfSync.** Fifteen of its eighteen resources are Synced; the
three `Permission` resources are not. The manifests declare `configure: ""`,
and the API server drops empty-string fields rather than storing them, so
ArgoCD compares three fields in Git against two in the cluster and reports a
diff that cannot converge.

The permission result is unaffected — RabbitMQ treats a missing `configure`
and an empty `configure` identically, both meaning no declare rights, which is
the intended least-privilege grant. The operator reports
`SuccessfulDeclare` and the Application is Healthy.

It is not purely cosmetic, though: with `selfHeal` enabled ArgoCD reapplies
the three resources roughly every five minutes, and the operator re-declares
them each time. The cluster load is negligible but the event log accumulates.
Removing the three `configure: ""` lines resolves it. That edit is deliberately
not in this change — it touches the message broker the demo depends on, and
the current behaviour is understood and harmless.

**No Ingress in `apps`.** Only `service/fastapi-ai-node-web` (ClusterIP,
8080) exists. The AI node is reached in-cluster and is not published through
an ALB; this is the intended topology, recorded here so its absence is not
later read as a missing resource.
