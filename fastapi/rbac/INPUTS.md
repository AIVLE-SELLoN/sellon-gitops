# FastAPI ServiceAccount and RBAC contracts and input gate

This directory owns one shared `ServiceAccount` (`fastapi-ai-node`) for all
four fastapi-ai-node workloads (`web`, `consumer`, `classification-worker`,
`daily-batch`). It is independently renderable — `kubectl kustomize
fastapi/rbac` does not depend on `fastapi/core`, `fastapi/batch`,
`fastapi/policies`, `apps/`, or the top-level kustomization — and, like
those sibling directories, is deliberately not wired into
`apps/fastapi.yaml` yet.

One ServiceAccount, not four: none of the four workloads need any
Kubernetes API permission (see below), so there is no least-privilege
reason to split by component the way `fastapi/policies`' NetworkPolicies
do. Splitting would only add operational surface (each Pod template having
to pick the right `serviceAccountName`) with no permission difference to
justify it.

## No Role/RoleBinding — this is deliberate, not an oversight

A Pod's `env[].valueFrom.secretKeyRef` / `envFrom[].configMapRef` (used
throughout `fastapi/core` for `LLM_API_KEY`, `MQ_USER`/`MQ_PASSWORD`,
`AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`, etc.) is resolved by the
kubelet when it starts the Pod — the container process never calls the
Kubernetes API to read that Secret itself. That means no workload here
needs `get`/`list`/`watch` on `secrets` (or anything else in the
Kubernetes API) via its own ServiceAccount, so no `Role`/`RoleBinding` was
created. Adding one "just in case" would be a standing permission with no
code path that ever uses it — the opposite of minimal.

If a future workload actually needs to call the Kubernetes API (e.g. to
read its own Pod metadata, list other resources, etc.), that specific,
justified need should get its own narrowly-scoped `Role`/`RoleBinding` at
that time — not added speculatively here.

## automountServiceAccountToken: false

Since nothing in this service calls the Kubernetes API, the ServiceAccount
token does not need to be mounted into any of these Pods at all. Set at the
`ServiceAccount` level so every workload referencing it inherits this by
default, without each Pod template needing to repeat
`automountServiceAccountToken: false` individually.

## IRSA — rejected (2026-08-17), not an open item

`00-serviceaccount.yaml` deliberately has no `eks.amazonaws.com/role-arn`
annotation, and this is a closed decision, not something left for future
review. Two independent reasons, either sufficient on its own:

1. The AI repo's `app/reporting/s3_uploader.py` passes static keys
   directly to its `boto3` client instead of using the default credential
   chain, and `ensure_s3_ready()` raises before upload if those keys are
   empty. An IRSA-issued role has no code path to plug into.
2. More fundamentally: a SigV4 presigned URL stays valid only as long as
   the credentials that signed it stay valid. IRSA's STS temporary
   credentials cap out at a 12-hour role session, which cannot produce the
   agreed 7-day presigned URL. This is not a code-level problem — no
   change to `s3_uploader.py` can make an STS session outlive itself.

Static keys are therefore the only mechanism that satisfies the 7-day
link. Daily's S3 auth is confirmed static-key (2026-08-17). Those keys are
supplied by the `fastapi-s3-credentials` Secret — ESO-synced from
`sellon/fastapi/s3` (the platform-owned source) — declared in
`fastapi/core/11-s3-external-secret.yaml` and owned by `fastapi/core`, not
created here. This directory only expects Deployments/CronJobs to
`secretKeyRef` into it by name, same as every other cross-directory Secret
reference already established in this service (`dockerhub-pull-secret`,
`fastapi-llm-credentials`, `ai-user-user-credentials`).

## Contract for other directories (not enforced here)

`fastapi/core`'s Deployments and `fastapi/batch`'s CronJobs should set
`spec.template.spec.serviceAccountName: fastapi-ai-node` once this
directory is merged alongside them — this repo does not edit those
sibling-branch files from here (out of scope), so this is recorded as a
contract to apply there, not a change made in this PR. Until a Pod
template sets `serviceAccountName` explicitly, it runs under the
namespace's `default` ServiceAccount instead, which is a gap worth
tracking, not a silent no-op.

`imagePullSecrets` wiring stays exactly where it already is — set directly
on each Pod template in `fastapi/core`/`fastapi/batch` (`dockerhub-pull-secret`).
It is not duplicated onto this ServiceAccount, to avoid having two
authoritative places for the same setting.

## Open items

None. This directory has no unresolved placeholders — the ServiceAccount
name and every field on it are fully specified.
