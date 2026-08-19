# Spring backend workload contracts and input gate

This directory owns only the Spring backend workload in `apps`. It does not
create the shared `ClusterSecretStore`, Docker Hub pull Secret, RabbitMQ User
CR, `apps` Namespace, or the future ArgoCD Application.

## Confirmed workload contracts

| Contract | Confirmed value | Evidence |
| --- | --- | --- |
| Namespace | `apps` | Deployment contract |
| Service and Pod label | `spring-backend`; `app: spring-backend` | Required by the Spring deployment and FastAPI PR 5's Daily NetworkPolicy |
| Container and Service port | `8080` | Spring Boot default; `server.port` is not configured |
| RabbitMQ host | `rabbitmq.default.svc.cluster.local:5672` | RabbitMQ runs in `default`; short DNS would resolve to `apps` |
| RabbitMQ credential Secret | `backend-user-user-credentials`, keys `username` / `password` | RabbitMQ User CR creates it in `apps`; no ExternalSecret is allowed for it |
| Image pull Secret | `dockerhub-pull-secret` | Shared `common/` resource; referenced only |
| Public host | `app.sellon.site` | `sellon.site` apex remains mail-only |

## application-prod.yaml environment contract

The ConfigMap supplies non-secret values and `spring-backend-credentials`
supplies the values below. `RABBITMQ_USERNAME` and `RABBITMQ_PASSWORD` are the
only exceptions: they come from the RabbitMQ Operator-generated Secret above.

| Environment variable | Source |
| --- | --- |
| `DB_URL` | ConfigMap |
| `DB_USERNAME`, `DB_PASSWORD` | `spring-backend-svc-db` (RDS-managed master user secret) |
| `REDIS_HOST`, `REDIS_PORT` | ConfigMap |
| `SMTP_USER`, `SMTP_PASSWORD` | `spring-backend-credentials` |
| `RABBITMQ_HOST`, `RABBITMQ_PORT` | ConfigMap |
| `SPRING_RABBITMQ_VIRTUAL_HOST` | ConfigMap |
| `RABBITMQ_USERNAME`, `RABBITMQ_PASSWORD` | `backend-user-user-credentials` |
| `AWS_ACCESS_KEY`, `AWS_SECRET_KEY` | `spring-backend-credentials` |
| `AWS_S3_REPORT_BUCKET`, `AWS_S3_IMAGE_BUCKET` | ConfigMap |
| `AWS_S3_REPORT_ACCESS_KEY`, `AWS_S3_REPORT_SECRET_KEY` | `spring-backend-credentials` |
| `AWS_S3_IMAGE_ACCESS_KEY`, `AWS_S3_IMAGE_SECRET_KEY` | `spring-backend-credentials` |
| `RAW_DB_URL` | ConfigMap |
| `RAW_DB_USERNAME`, `RAW_DB_PASSWORD` | `spring-backend-raw-db` (RDS-managed master user secret) |

`application-prod.yaml` does not set `spring.rabbitmq.virtual-host`, but the
project's queues are declared in the `app` vhost. Spring would default to `/`,
connect successfully, pass every probe, and receive no messages at all. The
ConfigMap therefore sets `SPRING_RABBITMQ_VIRTUAL_HOST`. Setting the property
in `application-prod.yaml` instead would be the cleaner fix; until then this
override is load-bearing and must not be removed.

`application.yaml` is also loaded under the `prod` profile. Its OpenSearch
configuration is enabled when `cloud.aws.opensearch.enabled` is absent, so the
manifest additionally supplies `AWS_OPENSEARCH_HOST`,
`AWS_OPENSEARCH_REGION`, `AWS_OPENSEARCH_ACCESS_KEY`, and
`AWS_OPENSEARCH_SECRET_KEY`. It does not silently disable that feature.

The same file also declares `cloud.aws.credentials.access-key` and
`secret-key` from `AWS_ACCESS_KEY` / `AWS_SECRET_KEY` with no default value,
so both are supplied as well. These are separate from the per-bucket S3 keys.

## Secrets Manager JSON key contract

`remoteRef.property` must exactly match a JSON key in Secrets Manager. The
following is a GitOps name-and-key contract, not evidence that the secret has
already been provisioned. No secret plaintext belongs in this repository.

| Proposed Secrets Manager source | Required JSON keys |
| --- | --- |
| `sellon/spring-backend/application` (to be created) | `SMTP_USER`, `SMTP_PASSWORD`, `AWS_ACCESS_KEY`, `AWS_SECRET_KEY`, `AWS_S3_REPORT_ACCESS_KEY`, `AWS_S3_REPORT_SECRET_KEY`, `AWS_S3_IMAGE_ACCESS_KEY`, `AWS_S3_IMAGE_SECRET_KEY`, `AWS_OPENSEARCH_ACCESS_KEY`, `AWS_OPENSEARCH_SECRET_KEY` |
| svc-db RDS-managed master user secret (exists) | `username`, `password` |
| raw-db RDS-managed master user secret (exists) | `username`, `password` |

Database credentials are deliberately not duplicated into the application
secret. `INFRA/data/rds.tf` provisions both instances with
`manage_master_user_password`, and raw-db carries an
`aws_secretsmanager_secret_rotation` resource, so a hand-maintained copy would
diverge from the live password at the first rotation. The connection URLs are
absent from those managed secrets, are not secret, and therefore live in the
ConfigMap; `application-prod.yaml` consumes `${DB_URL}` / `${RAW_DB_URL}` as a
single JDBC string, so no backend change is required.

## Activation blockers and unresolved inputs

| Required input | Current manifest value | Owner / required decision |
| --- | --- | --- |
| Spring image repository and immutable tag or digest | `<DOCKERHUB_NAMESPACE>/spring-backend:<IMMUTABLE_TAG_OR_DIGEST>` | Backend/Infra: publish a reachable image and replace the placeholder with an immutable tag or digest |
| OpenSearch domain does not exist | `<AWS_OPENSEARCH_HOST>` | Backend: no OpenSearch domain exists in this account. `OpenSearchConfig` is `@ConditionalOnProperty(matchIfMissing = true)` and its `@Value` bindings carry no defaults, so the bean is built and startup fails without a host. Either provision a domain or set `cloud.aws.opensearch.enabled: false`; the latter also removes the two OpenSearch keys from the Secrets Manager contract above |
| `KAFKA_BOOTSTRAP_SERVERS` unsupplied | absent from ConfigMap and ExternalSecret | Backend: `application-prod.yaml` resolves this with no default, so property binding fails before any probe runs. No Kafka usage exists in the Java sources, so `${KAFKA_BOOTSTRAP_SERVERS:}` is the smaller change; a real broker address can be added to the ConfigMap later without further manifest work |
| Secrets Manager provisioning | `sellon/spring-backend/application` contract only | Infra: create the secret with the documented JSON keys |
| ESO access to the application secret | not granted | Infra: `platform/irsa.tf` lists secret ARNs explicitly. `svc_db_secret_arn` and `raw_db_secret_arn` are already present, so the two database ExternalSecrets need no change, but the application secret's ARN must be added or every key it holds fails with AccessDenied |
| ALB DNS record | none in this repository | Infra: after ALB creation, create the Route 53 Alias for `app.sellon.site` as documented by `INFRA/platform/dns.tf` |
| Redis AUTH token | no password supplied | Data/Infra: `data/elasticache.tf` provisions a `redis_auth` secret, but neither `application-prod.yaml` nor this manifest carries a Redis password. If the cluster has an AUTH token enabled, a backend code change is required as well, not just a manifest key |
| JVM resources and container user | unset | Backend: provide measured requests/limits and verify the image user before setting `runAsNonRoot` |
| Ingress NetworkPolicy source restriction | ingress intentionally unrestricted | Infra: verify ALB traffic source/CIDRs before adding an ingress allow-list; an unverified selector can block the public API |
| Uploaded file durability | `file.storage-type: local`, `upload-dir: ./uploads` | Backend: uploads land on the container's ephemeral filesystem and are lost on restart. Acceptable for the demo; switching to the `s3` storage type needs the commented bucket settings in `application.yaml` |

## Resolved inputs

Recorded so a later reader can tell a confirmed value from an assumed one.

| Input | Resolved value | Evidence |
| --- | --- | --- |
| svc-db / raw-db endpoints | `sellon-svc-db...:5432/svcdb`, `sellon-raw-db...:5432/rawdb` | `INFRA/data` outputs `svc_db_endpoint` / `raw_db_endpoint`; database names fixed in `data/rds.tf` |
| svc-db / raw-db managed secret ARNs | the two `rds!db-...` ARNs referenced by the database ExternalSecrets | `INFRA/data` outputs `svc_db_secret_arn` / `raw_db_secret_arn`. RDS generates the six-character suffix, so the ARN is used rather than a guessed name |
| Redis endpoint | ElastiCache primary endpoint | `INFRA/data` output `redis_primary_endpoint`. The reader endpoint is unused: `application-prod.yaml` binds a single `spring.data.redis.host` |
| Report and image S3 bucket names | `sellon-reports-dev-...`, `sellon-images-dev-...` | Both buckets exist; the report bucket is console-managed by design and is not in Terraform |
| ACM certificate ARN | the certificate covering `app.sellon.site` | `INFRA/platform` output `acm_certificate_arn`, status ISSUED. Its SANs are `app.sellon.site` and `*.app.sellon.site` — note this is **not** a `*.sellon.site` wildcard, so any other second-level host would need a new certificate |

## Probe contract

`SecurityConfig.PERMIT_ALL_PATHS` permits `/actuator/health/**`, and
`application.yaml` limits management exposure to `health` with
`show-details: never` and probe groups enabled. Verified locally: all three
health paths return 200 without authentication, and other actuator endpoints
are unreachable.

Readiness and the startup probe use `/actuator/health/readiness`. Liveness
uses `/actuator/health/liveness` instead of the aggregate `/actuator/health`,
which reports DB, RabbitMQ, and Redis: a brief dependency outage would return
503 and restart an otherwise healthy JVM. The ALB health check targets the
readiness path so unready targets are drained rather than served. Do not point
any of these at an authenticated path; HTTP 401 would keep the Pod unready and
trigger liveness restarts.

The startup probe permits up to 150 seconds (5 seconds x 30 attempts) until
JVM cold-start timing is measured. It prevents liveness from restarting a
still-initializing Pod; tune the threshold after observing production starts.

## Cross-service boundary

FastAPI PR 5 already owns its Daily batch egress policy and permits TCP 8080
to Pods in `apps` selected by `app: spring-backend`. This directory honors
that label contract but does not edit `fastapi/`. NetworkPolicy cannot select
a Service by name, which is why the Pod label is a deployment-critical
contract rather than cosmetic metadata.

## Activation sequence

1. Terraform must have created the `apps` Namespace, ALB Controller, ESO,
   RabbitMQ Operator, and shared `ClusterSecretStore` / pull Secret.
2. Provision the unresolved endpoints, image, ACM ARN, and Secrets Manager
   JSON contract above; wait for `spring-backend-credentials` and
   `backend-user-user-credentials` in `apps`.
3. Render this directory and review the output. `apps/spring.yaml` is added by
   this change, so merging to `main` makes the root Application adopt it
   immediately; merge only after the blockers above are closed.
