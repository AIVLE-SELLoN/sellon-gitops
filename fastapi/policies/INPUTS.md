# FastAPI NetworkPolicy contracts and input gate

This directory owns egress `NetworkPolicy` resources for the four
fastapi-ai-node workloads only (`web`, `consumer`, `classification-worker`,
`daily-batch`). It is independently renderable — `kubectl kustomize
fastapi/policies` does not depend on `fastapi/core`, `fastapi/batch`,
`apps/`, or the top-level kustomization — and, like those two, is
deliberately not wired into `apps/fastapi.yaml` yet.

This branch (`feat/fastapi-network-operations`) was created from the
initial commit and does not contain `fastapi/core` or `fastapi/batch` —
those live on separate feature branches. The workload identity labels used
here (`app: fastapi-ai-node`, `app.kubernetes.io/name: fastapi-ai-node`,
`app.kubernetes.io/component: web|consumer|classification-worker|daily-batch`)
and the per-workload env contracts referenced below (which env each
workload reads, and therefore which egress it needs) are the same
conventions already established on those sibling branches — carried over
from that shared project context, not re-derived or guessed here, since
this branch has no local files to read them from.

## Namespace boundary — why every cross-namespace rule has a namespaceSelector

RabbitMQ and ChromaDB run in `default`; kube-dns runs in `kube-system`; the
fastapi workloads themselves run in `apps`. A NetworkPolicy `to:`/`from:`
peer with a `podSelector` but no `namespaceSelector` matches only pods in
the **same namespace as the NetworkPolicy itself**. Every rule in this
directory that targets `default` or `kube-system` sets an explicit
`namespaceSelector` (using the automatic `kubernetes.io/metadata.name`
namespace label, present since Kubernetes 1.21) for exactly this reason —
without it, the rule would silently scope to `apps` only, match nothing,
and block that traffic with no error pointing at the missing selector.

## Un-pinned Pod-level selectors (RabbitMQ, ChromaDB)

RabbitMQ's and ChromaDB's actual Pod labels are owned by the platform team
outside this repo and are not confirmed here. Rather than guess a
podSelector label that might be wrong (which would silently break
connectivity the same way a missing namespaceSelector would), the
RabbitMQ/ChromaDB egress rules are scoped by `namespaceSelector` (=
`default`) + port only. This is a real, working policy today — not a
placeholder — but it is broader than a fully pod-scoped rule would be.

| Follow-up (optional hardening) | Needs |
| --- | --- |
| Add `podSelector` to the RabbitMQ egress rules (Consumer) | Confirmed Pod label for the RabbitMQ deployment in `default` |
| Add `podSelector` to the ChromaDB egress rules (Web, Consumer, Daily) | Confirmed Pod label for the ChromaDB deployment in `default` |

## External domain limitation (LLM, S3)

Kubernetes `NetworkPolicy` operates at L3/L4 (IP address + port) only. It
has no field for domain name, hostname, or SNI/TLS-based matching, so
"allow egress to `api.openai.com` (or whichever LLM provider) and nothing
else" is not something a vanilla `NetworkPolicy` can express — SaaS LLM
providers do not publish stable, narrow IP ranges to pin an `ipBlock` to,
and even AWS S3's published IP ranges are large and change over time.

The external-HTTPS rules in this directory therefore allow
`ipBlock: 0.0.0.0/0` minus the RFC1918 private ranges
(`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`), restricted to port 443.
This is deliberately not scoped to "just LLM" or "just S3" — it is "any
external host, port 443 only." The RFC1918 exclusion exists so this rule
doesn't also grant the pod arbitrary access to internal VPC/cluster
services listening on 443; it does not, and cannot, narrow the rule down to
the specific SaaS endpoints these workloads actually call.

Achieving true domain-level egress filtering would require something
outside plain `NetworkPolicy` — e.g. an egress proxy/gateway the workloads
are forced through, a service mesh or CNI with FQDN-aware policies (Istio
egress rules, Cilium `toFQDNs`), or a VPC-layer domain-filtering firewall
(AWS Network Firewall). None of that exists in this repo; this limitation
is inherent to the primitive being used here, not a gap specific to these
four manifests.

## spring-backend egress (Daily only)

`04-daily-batch-networkpolicy.yaml` allows Daily → `spring-backend` on port
8080, selected by `podSelector: matchLabels: {app: spring-backend}` in the
same `apps` namespace (no `namespaceSelector`, since same-namespace is the
correct/intended scope for this one rule). `NetworkPolicy` cannot select by
Service name, so this had to be a Pod label, not the backend Service's
name. The Spring manifests themselves do not exist in this repo — this is
a forward reference to a label contract owned by the Spring backend's own
service directory, not a resource created here.

This exists for Daily's `GET /internal/alerts/active?since=35d` call: the
backend provides that endpoint and Daily is the caller, so it is an
in-cluster path, not part of the external-HTTPS-443 rule above.

## Redis / ElastiCache

Redis is unused — 6379 is not opened anywhere in this directory. Every
raw-PostgreSQL egress rule pins the port to `5432` specifically (not left
as a bare CIDR-only rule) because ElastiCache shares the same data subnets
as the RDS instance; without the port restriction, a CIDR-only rule to
those subnets would also reach 6379.

## Data subnet CIDRs kept separate

`10.0.48.0/24` (2a) and `10.0.49.0/24` (2c) are each their own `ipBlock`
entry in every raw-PostgreSQL egress rule, not merged into `10.0.48.0/23`.
Merging into the /23 supernet would silently include a third subnet if one
is ever added inside that range later; two explicit /24s only ever mean
exactly these two subnets.

## Resolved: Daily needs RabbitMQ 5672 egress too

This task's instruction scoped RabbitMQ 5672 egress to **Consumer only**.
But the sibling `feat/fastapi-batch-jobs` branch wires
`MQ_HOST`/`MQ_USER`/`MQ_PASSWORD`/`MQ_ENABLED=true` into Daily's env, and
the AI repo's `app/batch/daily.py` calls `publish_anomaly_analyzed()` and
`publish_guideline_generated()` from `app.core.mq` — Daily is a second
publisher on the same exchange Consumer reads from, not just a reader of
other services. `04-daily-batch-networkpolicy.yaml` now grants Daily the
same `default`-namespace 5672 egress as Consumer.

## Open items — do not resolve arbitrarily

| Required input | Status | Owner |
| --- | --- | --- |
| RabbitMQ Pod label (for tightening the Consumer/Daily egress rules) | Not confirmed from this repo | Platform |
| ChromaDB Pod label (for tightening Web/Consumer/Daily egress rules) | Not confirmed from this repo | Platform |
| `spring-backend` Pod label contract (`app: spring-backend`) | Confirmed by this task's instruction; not yet backed by an actual Spring manifest in this repo | Backend (Spring team) |
