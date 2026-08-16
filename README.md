# sellon-gitops

SELLoN 클러스터에서 ArgoCD가 관리하는 Kubernetes 매니페스트의 정본 저장소입니다.

> ArgoCD 설치와 root Application 생성은 `INFRA/platform`이 담당합니다.
> 이 저장소는 root Application이 읽는 `apps/`와 서비스별 매니페스트만 관리합니다.
> `gitops_repo_url`이 비어 있는 동안에는 ArgoCD만 설치되고 이 저장소는 동기화되지
> 않습니다. 비공개 저장소 인증정보는 Git에 저장하지 않습니다.

## 왜 앱 저장소와 분리하는가

- CI가 자기 저장소에 커밋하면서 빌드 루프를 도는 사고를 구조적으로 막습니다
- 배포 이력이 이 저장소의 커밋 로그로 남아 "언제 무엇이 배포됐는지"가 명확합니다
- 앱 코드 권한과 배포 권한을 분리할 수 있습니다

## 구조

```text
sellon-gitops/
├── apps/          ArgoCD Application 정의 (root Application이 recurse로 읽음)
│   ├── README.md
│   └── rabbitmq.yaml
└── rabbitmq/      RabbitMQ 클러스터 + 메시징 토폴로지
    ├── kustomization.yaml
    ├── 00-cluster.yaml      RabbitmqCluster
    ├── 10-vhost.yaml        Vhost
    ├── 20-exchanges.yaml    app.events, app.events.dlx
    ├── 30-queues.yaml       main.inbound, ai.inbound, app.events.dead
    ├── 40-bindings.yaml     라우팅 4종
    └── 50-users.yaml        User + Permission
```

새 서비스는 `apps/<service>.yaml`과 `<service>/`를 한 쌍으로 추가합니다. 서비스별
디렉터리는 해당 서비스가 소유하며 다른 서비스의 리소스를 포함하거나 패치하지 않습니다.
공통 규칙 변경과 서비스 워크로드 변경은 가능한 한 별도 PR로 분리합니다.

현재 등록된 서비스는 RabbitMQ뿐입니다. FastAPI는 후속 PR에서 `fastapi/`를 작성한 뒤
활성화 단계에서 `apps/fastapi.yaml`을 추가합니다. 그 외 서비스 목록은 아직 미확정입니다.

## 역할 경계 — 무엇이 Terraform이고 무엇이 여기인가

| | 담당 | 이유 |
|---|---|---|
| EKS, 노드그룹, 애드온 | Terraform (`platform/`) | AWS 리소스 |
| cert-manager, RabbitMQ 오퍼레이터, ArgoCD | Terraform (`platform/`) | 부트스트랩. ArgoCD가 없어도 서야 함 |
| **커스텀 리소스 전부** | **여기 (ArgoCD)** | 아래 참고 |

커스텀 리소스를 Terraform에 두지 않는 이유: `kubernetes_manifest` 는 plan 시점에
대상 CRD가 클러스터에 이미 존재해야 합니다. 오퍼레이터와 그 CR을 같은 스택에 두면
최초 apply 때 CRD가 아직 없어 plan 자체가 실패합니다 — 순환이라 우회가 없습니다.

## 선행 조건

이 저장소의 매니페스트가 동작하려면 platform 스택이 먼저 apply되어 있어야 합니다.

1. `bootstrap` → `network` → `data` → `platform` 순서로 apply
2. platform 스택이 cert-manager, RabbitMQ 오퍼레이터 2종, ArgoCD를 설치
3. `gitops_repo_url` 을 채우고 다시 apply → root Application 생성
4. root Application이 `apps/` 를 읽고 나머지를 동기화

## 적용 순서 (sync-wave)

토폴로지 리소스는 서로 의존하므로 순서가 중요합니다.

| wave | 리소스 | 왜 이 순서인가 |
|---|---|---|
| 0 | `RabbitmqCluster` | 클러스터가 Ready 되어야 오퍼레이터가 접속해 나머지를 만들 수 있음 |
| 1 | `Vhost` | 모든 토폴로지 리소스가 담기는 컨테이너 |
| 2 | `Exchange`, `User` | vhost만 있으면 생성 가능 |
| 3 | `Queue`, `Permission` | Queue의 DLX 인자가 가리키는 Exchange가 먼저 있어야 함 |
| 4 | `Binding` | source Exchange와 destination Queue가 모두 필요 |

클러스터 기동이 느리면 첫 동기화가 실패할 수 있어서 Application에 재시도를
넉넉히(10회, 15s→5m 백오프) 걸어두었습니다.

### health check는 ArgoCD에 내장되어 있습니다

별도 설정이 필요 없습니다. ArgoCD는 `resource_customizations/rabbitmq.com` 아래에
**우리가 쓰는 kind 전부**의 health check를 내장하고 있습니다 (v3.5.1 기준 확인):
`Binding` / `Exchange` / `Permission` / `Policy` / `Queue` / `RabbitmqCluster` /
`Shovel` / `User` / `Vhost`.

`Queue/health.lua` 는 `status.conditions` 의 `Ready` 조건을 읽습니다.

| CR 상태 | ArgoCD 판정 |
|---|---|
| `Ready=True` + `SuccessfulCreateOrUpdate` | `Healthy` |
| `Ready=False` + `FailedCreateOrUpdate` | `Degraded` |
| 그 외 (아직 처리 전 등) | `Unknown` |

Application 의 health 는 하위 리소스 중 **가장 나쁜 것**을 따릅니다
(gitops-engine 의 `healthOrder`: Healthy → Suspended → Progressing → Missing →
Degraded → Unknown). 그래서 큐 하나가 실패하면 Application 도 `Degraded` 입니다.

> **"ArgoCD는 초록인데 RabbitMQ는 실패" 상태는 생기지 않습니다.**
> 단, **Sync 상태와 Health 상태는 별개**입니다. `Sync: Synced` 는 "매니페스트를
> 적용했다"까지만 뜻하고, 실제 성공 여부는 **Health 열**이 알려줍니다.
> 대시보드에서 Sync 가 아니라 Health 를 보세요.

직접 확인하고 싶다면:

```bash
kubectl get queues.rabbitmq.com,bindings.rabbitmq.com,exchanges.rabbitmq.com \
  -n default -o custom-columns=KIND:.kind,NAME:.metadata.name,READY:.status.conditions[0].status
```

## 저장소 주소가 두 곳에 있습니다

`apps/rabbitmq.yaml` 의 `repoURL` / `targetRevision` 은 하드코딩이고,
root Application은 `platform/terraform.tfvars` 의 `gitops_repo_url` /
`gitops_repo_branch` 로 만들어집니다. **두 값이 어긋나면 root는 정상 동기화되고
Healthy로 보이는데 `rabbitmq` Application만 `ComparisonError` 에 머물러
토폴로지가 하나도 생성되지 않습니다.**

저장소 이름·조직·브랜치를 바꿀 때 확인할 곳:

1. `platform/terraform.tfvars` — `gitops_repo_url`, `gitops_repo_branch`
2. `apps/rabbitmq.yaml` — `spec.source.repoURL`, `spec.source.targetRevision`

저장소가 **비공개면** ArgoCD에 접근 자격증명(repo Secret)을 별도로 등록해야
합니다. 현재 `platform/argocd.tf` 는 아무 자격증명도 만들지 않습니다.
현재 인증 상태와 최초 연결 순서는
[`docs/argocd-private-repository-auth.md`](docs/argocd-private-repository-auth.md)를
참고하세요.

## 삭제 보호

모든 `Queue` 와 `Vhost` 에 `deletionPolicy: retain` 을 걸어두었습니다.

CRD 기본값은 `delete` 이고 Application에 `prune: true` 가 걸려 있어서, 기본값
그대로면 **`30-queues.yaml` 을 지우거나 이름을 바꾸거나 `kustomization.yaml`
에서 빼는 것만으로** 오퍼레이터가 실제 `queue.delete` 를 발행합니다. 큐 안의
메시지는 그대로 사라지고 복구 경로가 없습니다 — 사후 분석용으로 TTL까지 뺀
`app.events.dead` 가 특히 그렇습니다.

`retain` 이면 CR만 사라지고 브로커의 큐와 메시지는 남습니다. 큐를 정말 없앨
때는 수동으로 지워야 하지만, 실수로 지우는 것보다 나은 거래입니다.

**`deletionPolicy` 는 모든 종류에 있는 게 아닙니다.** 오퍼레이터 v1.20.1에서 이
필드를 가진 건 `Federation` / `Queue` / `Shovel` / `Vhost` 4종뿐입니다.
`Exchange` 에는 없어서 `RabbitmqCluster` 와 함께 **ArgoCD 어노테이션**으로
보호합니다:

```yaml
argocd.argoproj.io/sync-options: Prune=false,Delete=false
```

두 옵션이 서로 다른 경로를 막으므로 **둘 다** 필요합니다.

| 옵션 | 막는 경로 |
|---|---|
| `Prune=false` | 이 Application을 sync할 때, 파일이 없어졌다고 리소스를 지우는 것 |
| `Delete=false` | 이 Application 자체가 삭제될 때의 cascade 삭제 |

후자가 더 현실적인 경로입니다. `apps/rabbitmq.yaml` 을 지우거나 이름을 바꾸면
root Application(`prune: true`)이 `rabbitmq` Application을 prune하고, 거기 붙은
`resources-finalizer.argocd.argoproj.io` 가 관리 대상을 연쇄 삭제합니다.
`RabbitmqCluster` 가 이 경로로 사라지면 StatefulSet과 PVC까지 가고, gp3
StorageClass가 `reclaim_policy = "Delete"` 라 **EBS 볼륨 자체가 삭제**됩니다.
그 시점엔 큐의 `deletionPolicy: retain` 도 무의미합니다 — 데이터가 있던 디스크가
없어졌으니까요.

DLX 삭제가 특히 위험합니다. `app.events` 가 사라지면 발행자가 채널 에러를 받아
바로 티가 나지만, **`app.events.dlx` 가 사라지면 RabbitMQ는 dead letter를 그냥
버립니다** — 재시도 한도를 넘긴 것도, TTL이 지난 것도 조용히 사라지고
`app.events.dead` 는 텅 빈 채로 아무 에러도 안 납니다.

`Binding` / `User` / `Permission` 은 **일부러 보호하지 않았습니다.** 바인딩은
라우팅 설정이라 운영 중 정상적으로 추가·제거되는 대상인데, 여기에 `Prune=false`
를 걸면 저장소에서 지워도 브로커에 남아 GitOps 상태와 실제가 갈라집니다.
보호는 "데이터를 담는 것"(클러스터·vhost·큐)과 "없어지면 조용히 실패하는
것"(익스체인지)에만 겁니다.

## 애플리케이션 접속 정보

`User` CRD가 비밀번호를 자동 생성해 Secret에 넣습니다.

```bash
kubectl -n default get secret backend-user-user-credentials \
  -o jsonpath='{.data.password}' | base64 -d
```

연결 URI:

```
amqp://<user>:<pass>@rabbitmq.default.svc.cluster.local:5672/app
```

vhost 이름을 `/app` 이 아니라 `app` 으로 정했습니다. 슬래시가 이름에 들어가면
URI에서 `%2Fapp` 으로 인코딩해야 하고, 이걸 놓치면 다른 vhost를 가리켜
`ACCESS_REFUSED` 가 납니다. 아직 배포된 것이 없어 바꾸는 비용이 0이라 함정
자체를 없앴습니다. **Notion CRD 문서의 `vhost: /app` 값도 함께 갱신해야 합니다.**

### Secret이 앱과 같은 네임스페이스에 생기는 이유

`backend-user` / `ai-user` 의 `User` CR은 **`apps` 네임스페이스**에 있습니다.
operator가 자격증명 Secret을 `user.Namespace` 에 만들기 때문에, CR을 `apps` 에
두면 Secret도 거기 생깁니다. 앱 파드가 바로 읽을 수 있어서 **PushSecret 이나
Reflector 같은 복사 장치가 필요 없습니다.**

Secret은 네임스페이스 스코프라, CR을 `default` 에 두면 `apps` 의 파드가 읽지
못합니다. 그래서 계정만 앱 쪽으로 뺐습니다.

이 배치가 성립하려면 두 가지가 맞아야 합니다.

1. `00-cluster.yaml` 의 `RabbitmqCluster` 에 다음 어노테이션이 있어야 합니다.
   없으면 operator가 교차 네임스페이스 참조를 거부합니다.

   ```yaml
   rabbitmq.com/topology-allowed-namespaces: "apps"
   ```

2. `apps` 네임스페이스가 먼저 존재해야 합니다. `platform/k8s_namespaces.tf` 가
   만들고, Terraform이 ArgoCD보다 먼저 돌기 때문에 순서는 자연히 맞습니다.

`dlq-reprocessor-user` 는 앱이 아니라 운영자가 쓰는 계정이라 `default` 에 그대로
둡니다. 클러스터와 같은 네임스페이스라 어노테이션도 필요 없습니다.

> 네임스페이스 이름을 바꾸려면 세 곳을 함께 고쳐야 합니다 —
> `platform/terraform.tfvars` 의 `apps_namespace` / `chromadb_client_namespace`,
> `00-cluster.yaml` 의 어노테이션, `50-users.yaml` 의 네임스페이스.

## CRD 문서 대비 추가된 것

Notion「CRD」문서에 정의가 빠져 있어 여기서 채운 오브젝트입니다. 참조만 되고
실체가 없으면 적용이 실패하거나 메시지가 조용히 유실됩니다.

| 오브젝트 | 문제 |
|---|---|
| `app.events` Exchange | 두 Binding의 source인데 정의가 없었음 → Binding이 NOT_FOUND로 실패 |
| `ai.inbound` Queue | "기존 큐"로 표기됐지만 그 기존은 AI팀 로컬 브로커 기준. 실제 클러스터에는 없음 |
| `ai.inbound` 의 DLX 인자 | dead 큐와 바인딩은 있는데 정작 원본 큐에서 그리로 보내는 설정이 없었음 |
| `app` Vhost | 모든 리소스가 쓰는데 만드는 정의가 없었음 |
| `User` / `Permission` | 앱 접속 계정이 없었음. guest는 loopback 전용이라 파드에서 못 씀 |

## CRD 문서와 의도적으로 다른 점

| 항목 | 문서 | 여기 | 이유 |
|---|---|---|---|
| vhost 이름 | `/app` | `app` | URI에서 `%2F` 인코딩 함정 제거 |
| DLQ 개수 | `main.inbound.dead` / `ai.inbound.dead` 2개 | `app.events.dead` 1개 | 운영 방침이 DLQ 단일화 |

**두 항목 모두 Notion CRD 문서 갱신이 필요합니다.**

DLQ를 합쳐도 출처는 구분됩니다. 원본 큐마다 `x-dead-letter-routing-key` 를
`main.inbound.dead` / `ai.inbound.dead` 로 다르게 유지했고, RabbitMQ가 dead letter
시 붙이는 `x-death` 헤더에 원본 큐 이름·사유(`rejected` / `expired` /
`delivery_limit`)·횟수·원래 라우팅 키가 들어갑니다. 재처리 로직은 이걸 보고
분기하면 됩니다.

## 남은 결정 사항

- **DLQ 보존 기간과 정리 주체** — `app.events.dead` 에 TTL을 넣지 않았습니다. 대신
  `x-max-length-bytes` 1GiB + `x-overflow: drop-head` 로 임시 상한만 걸어두었습니다.
  상한이 없으면 노드당 5Gi PVC(모든 큐 공유)가 차고, `disk_free_limit` 알람이
  걸리는 순간 브로커가 **살아있는 큐까지 포함해 모든 발행을 차단**합니다.
  보존 정책이 정해지면 TTL이나 정기 배출로 대체하는 것이 맞습니다
- **AI팀 권한 확정** — 최소 권한으로 두어 AI 쪽 바인딩 코드는 동작하지 않습니다. 전달 필요
- **`passive declare` 동작 확인** — AI팀이 쓰는 존재 확인 방식이 현재 권한으로 통과하는지
## 이 구성이 견디는 장애의 범위

노드를 3대로 늘리고 anti-affinity를 `required` 로 건 것은 **노드 장애 내성**까지
사는 것입니다. **AZ 장애 내성은 사지 못했습니다.**

| 장애 | 결과 |
|---|---|
| 노드 1대 사망 | ✅ 2/3 유지, quorum 정상, 서비스 계속 |
| **AZ-a 전체 장애** | ❌ 3중 2 상실, quorum 붕괴, 모든 quorum 큐 쓰기 중단 |

network 스택의 AZ가 `ap-northeast-2a` / `2c` 두 개뿐이라 노드 3대는 AZ 기준
**2:1** 로 배치되고, `topologyKey: kubernetes.io/hostname` 은 호스트 단위 분산만
보장하기 때문입니다. 여기에 NAT Gateway도 AZ-a에 1개뿐이고 `rt-app-a` /
`rt-app-c` 가 둘 다 그것을 가리켜서(`network/vpc.tf`), AZ-a 장애 시 quorum이
깨지는 동시에 살아남은 AZ-c 노드의 아웃바운드까지 끊깁니다.

**의도적인 선택입니다.** AZ 이중화를 하려면 network 스택에 세 번째 AZ와 서브넷
3세트를 추가하고 NAT도 AZ별로 둬야 하는데(NAT 비용 3배, 주당 +$20 수준), 노드
한 대 추가($9.2/주)보다 비싸고 시연·테스트 규모에서는 사는 값에 비해 과합니다.

다른 스택은 노드 수를 참조하지 않으므로 이 결정으로 바꿔야 할 코드는 없습니다.
app 서브넷이 `/20` 두 개(각 4,096 IP)라 prefix delegation을 켠 노드 3대(파드
최대 330개)에도 주소 여유가 충분합니다.

## 노드 수와 replicas는 함께 움직입니다

`00-cluster.yaml` 의 `replicas: 3` 과 `platform/variables.tf` 의
`node_desired_size = 3` 은 **한 쌍입니다.** anti-affinity가 `required` 라서,
노드가 3대 미만이면 남는 RabbitMQ 파드가 영구 `Pending` 에 빠집니다.

노드를 2대로 되돌리려면 `replicas` 를 낮추고 anti-affinity도 `preferred` 로
내려야 합니다. 그 경우 quorum 내성은 포기하는 것이므로, CRD 문서의
`type: quorum` 을 `classic` 으로 바꾸고 quorum 전용 인자인 `x-delivery-limit`
도 함께 빼는 것이 정직한 구성입니다.

메모리 `limits` 는 알람 임계를 직접 결정합니다. cluster operator가 이 값으로
`total_memory_available_override_value` 를 세우고, RabbitMQ가 거기에
`vm_memory_high_watermark` 비율을 곱한 지점에서 알람을 겁니다. 알람이 걸린
노드는 **모든 발행을 차단**하므로 백엔드까지 멈춥니다. 낮춰 잡지 마세요.

비율은 버전마다 다릅니다 — RabbitMQ 4.x 기본값은 `0.6`, 그 이전은 `0.4` 입니다.
`spec.image` 를 비워 버전을 오퍼레이터에 맡기고 있으므로 특정 수치를 여기 못박지
않습니다. 배포 후 실제 값으로 확인하세요:

```bash
kubectl -n default exec rabbitmq-server-0 -- \
  rabbitmq-diagnostics status | grep -i "memory high watermark"
```
