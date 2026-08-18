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
| `DB_URL`, `DB_USERNAME`, `DB_PASSWORD` | `spring-backend-credentials` |
| `REDIS_HOST`, `REDIS_PORT` | ConfigMap |
| `SMTP_USER`, `SMTP_PASSWORD` | `spring-backend-credentials` |
| `RABBITMQ_HOST`, `RABBITMQ_PORT` | ConfigMap |
| `SPRING_RABBITMQ_VIRTUAL_HOST` | ConfigMap |
| `RABBITMQ_USERNAME`, `RABBITMQ_PASSWORD` | `backend-user-user-credentials` |
| `AWS_ACCESS_KEY`, `AWS_SECRET_KEY` | `spring-backend-credentials` |
| `AWS_S3_REPORT_BUCKET`, `AWS_S3_IMAGE_BUCKET` | ConfigMap |
| `AWS_S3_REPORT_ACCESS_KEY`, `AWS_S3_REPORT_SECRET_KEY` | `spring-backend-credentials` |
| `AWS_S3_IMAGE_ACCESS_KEY`, `AWS_S3_IMAGE_SECRET_KEY` | `spring-backend-credentials` |
| `RAW_DB_URL`, `RAW_DB_USERNAME`, `RAW_DB_PASSWORD` | `spring-backend-credentials` |

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
| `sellon/spring-backend/application` | `DB_URL`, `DB_USERNAME`, `DB_PASSWORD`, `SMTP_USER`, `SMTP_PASSWORD`, `AWS_ACCESS_KEY`, `AWS_SECRET_KEY`, `AWS_S3_REPORT_ACCESS_KEY`, `AWS_S3_REPORT_SECRET_KEY`, `AWS_S3_IMAGE_ACCESS_KEY`, `AWS_S3_IMAGE_SECRET_KEY`, `RAW_DB_URL`, `RAW_DB_USERNAME`, `RAW_DB_PASSWORD`, `AWS_OPENSEARCH_ACCESS_KEY`, `AWS_OPENSEARCH_SECRET_KEY` |

## Activation blockers and unresolved inputs

| Required input | Current manifest value | Owner / required decision |
| --- | --- | --- |
| Spring image repository and immutable tag or digest | `<DOCKERHUB_NAMESPACE>/spring-backend:<IMMUTABLE_TAG_OR_DIGEST>` | Backend/Infra: publish a reachable image and replace the placeholder with an immutable tag or digest |
| Redis endpoint | `<REDIS_HOST>` | Data/Infra: confirm the production ElastiCache DNS name |
| Report and image S3 bucket names | `<AWS_S3_REPORT_BUCKET>`, `<AWS_S3_IMAGE_BUCKET>` | Backend/Infra: confirm the two bucket names |
| OpenSearch endpoint | `<AWS_OPENSEARCH_HOST>` | Backend/Infra: confirm its host and whether it is public or VPC-private |
| Secrets Manager provisioning | `sellon/spring-backend/application` contract only | Infra: create the secret with every documented JSON key and permit ESO access |
| ACM certificate ARN | `<ACM_CERTIFICATE_ARN>` | Infra: provide `INFRA/platform` output `acm_certificate_arn`; retain the existing certificate, do not create a new one |
| ALB DNS record | none in this repository | Infra: after ALB creation, create the Route 53 Alias for `app.sellon.site` as documented by `INFRA/platform/dns.tf` |
| JVM resources and container user | unset | Backend: provide measured requests/limits and verify the image user before setting `runAsNonRoot` |
| Ingress NetworkPolicy source restriction | ingress intentionally unrestricted | Infra: verify ALB traffic source/CIDRs before adding an ingress allow-list; an unverified selector can block the public API |
| Redis AUTH token | no password supplied | Data/Infra: `data/elasticache.tf` provisions a `redis_auth` secret, but neither `application-prod.yaml` nor this manifest carries a Redis password. If the cluster has an AUTH token enabled, a backend code change is required as well, not just a manifest key |
| Uploaded file durability | `file.storage-type: local`, `upload-dir: ./uploads` | Backend: uploads land on the container's ephemeral filesystem and are lost on restart. Acceptable for the demo; switching to the `s3` storage type needs the commented bucket settings in `application.yaml` |

## Probe contract

No actuator endpoint is assumed. `SecurityConfig.PERMIT_ALL_PATHS` permits
`/v3/api-docs/**`, and the Springdoc UI dependency is present, so the
Deployment uses unauthenticated `GET /v3/api-docs` for readiness and liveness.
The same path is configured as the ALB target health check: the default `/`
path is authenticated and would leave every target unhealthy. Do not replace
it with an authenticated API endpoint: HTTP 401 would keep the Pod unready,
cause liveness restarts, or drain it from the ALB.

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
3. Render this directory and review the output. This change does not add
   `apps/spring.yaml`; ArgoCD activation is a separate step.
