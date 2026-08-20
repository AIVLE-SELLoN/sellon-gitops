# fastapi/core rendering verification

`kubectl kustomize fastapi/core` renders cleanly with no errors and produces
7 resources: 2 ConfigMap, 1 Service, 2 Deployment, 2 ExternalSecret.

## Checked against the PR 3 commit-2 contract

- [x] `namespace: apps` on every resource
- [x] Labels `app=fastapi-ai-node`, `app.kubernetes.io/name=fastapi-ai-node`
      applied to every resource via the base `labels` block
- [x] `imagePullSecrets: [dockerhub-pull-secret]` referenced by name only,
      not created here
- [x] Image carries an immutable `main-<shortSHA>` tag, not `latest`. The
      concrete value is set centrally in `fastapi/kustomization.yaml`, which
      overrides the reference in these files; the repository is public, so no
      pull credential is required for it (`dockerhub-pull-secret` remains for
      rate-limit headroom)
- [x] Web: `containerPort` and Service `targetPort` both `8080`;
      `GET /health` on readiness and liveness
- [x] Web/Consumer resource requests/limits match the confirmed contract
      (100m/256Mi–500m/512Mi web; 50m/128Mi–250m/256Mi consumer)
- [x] `MQ_HOST`/`CHROMA_HOST` resolve to `default` namespace cluster FQDNs,
      not `apps`
- [x] `CHROMA_PERSIST_DIR` is absent from both ConfigMaps
- [x] Secret references match the confirmed contract:
      `fastapi-llm-credentials` (LLM), `ai-user-user-credentials` (RabbitMQ),
      `fastapi-s3-credentials` ExternalSecret present for the future Daily
      workload (PR 4)
- [x] No `RAW_DB_PATH` / SQLite value anywhere in these manifests

## Input gates still open (see INPUTS.md)

- `MQ_COMPANY_ID` real value — Backend

The Docker Hub namespace is confirmed as `y0njunch0i`. For the deployed
state of this directory as observed on the cluster, see `fastapi
/VERIFICATION.md`; this file records the pre-apply rendering review only.
