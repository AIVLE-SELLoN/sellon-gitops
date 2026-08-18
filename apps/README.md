# ArgoCD Application 규칙

`INFRA/platform`이 생성하는 root Application은 이 디렉터리를 재귀적으로 읽습니다.
여기에는 서비스별 ArgoCD `Application`만 두고 Deployment, Service, Secret 같은
워크로드 리소스는 두지 않습니다.

## 책임 경계

| 위치 | 담당 |
| --- | --- |
| `INFRA/platform` | ArgoCD 설치와 root Application 정의, EKS·IAM 등 기반 리소스 |
| `sellon-gitops/apps/<service>.yaml` | 서비스별 ArgoCD Application |
| `sellon-gitops/<service>/` | 해당 서비스의 Kubernetes 매니페스트 |
| `sellon-gitops/common/` | 여러 서비스가 참조만 하는 공용 Kubernetes 리소스 |

root Application은 `INFRA/platform`에만 두며 이 저장소에서 중복 생성하지 않습니다.
반대로 서비스별 Application과 워크로드 매니페스트는 Terraform에서 생성하지 않습니다.

## 디렉터리 계약

- Application 파일: `apps/<service>.yaml`
- 매니페스트 경로: `<service>/`
- `spec.source.path`: 서비스 디렉터리와 동일한 이름
- `spec.source.repoURL`: 이 저장소의 URL
- `spec.source.targetRevision`: root Application과 동일한 브랜치
- `spec.destination.namespace`: 서비스가 실제 리소스를 생성할 namespace

각 서비스는 자기 디렉터리만 소유합니다. 다른 서비스의 리소스를 참조해야 할 때는
DNS, Secret 이름, label 같은 계약만 문서화하고 상대 서비스의 YAML을 직접 수정하지
않습니다.

`common/`은 서비스 디렉터리가 소유할 수 없는 공용 리소스 전용입니다. 현재
`apps/common.yaml`은 공용 ESO `ClusterSecretStore`와 Docker Hub pull
ExternalSecret만 동기화합니다. 서비스별 LLM 등 ExternalSecret은 이 경로에 추가하지
않고 해당 서비스 PR에서 작성하며, 서비스는 공용 Store와 Secret 이름을 참조만 합니다.

## Application 공통 계약

현재 저장소와 `apps/rabbitmq.yaml`을 기준으로 다음 값을 공통 기본값으로 사용합니다.

| 필드 | 공통 규칙 |
| --- | --- |
| `metadata.name` | `<service>` |
| `metadata.namespace` | `argocd` |
| `spec.project` | `default` |
| `spec.source.repoURL` | `https://github.com/AIVLE-SELLoN/sellon-gitops.git` |
| `spec.source.targetRevision` | `main` |
| `spec.source.path` | `<service>`이며 실제 디렉터리가 존재해야 함 |
| `spec.destination.server` | `https://kubernetes.default.svc` |
| `spec.destination.namespace` | 서비스 계약에서 확정한 namespace |
| `spec.syncPolicy.automated.prune` | 기본 `true` |
| `spec.syncPolicy.automated.selfHeal` | 기본 `true` |

`repoURL`과 `targetRevision`은 platform root Application의 `gitops_repo_url`,
`gitops_repo_branch`와 반드시 같아야 합니다. 이 저장소의 현재 계약은 위 URL과
`main`이지만, 실제 platform 입력 및 apply 여부는 별도로 확인해야 합니다.

`destination.namespace`는 공통값으로 추정하지 않습니다. Application을 추가하는 PR에서
서비스의 namespace와 생성 주체를 확인하고 명시합니다. 현재 확정 상태는 다음과 같습니다.

| 서비스 | Application | path | destination namespace | 상태 |
| --- | --- | --- | --- | --- |
| RabbitMQ | `apps/rabbitmq.yaml` | `rabbitmq` | `default` | 등록됨 |
| 공용 ESO/이미지 pull | `apps/common.yaml` | `common` | `default` (명시된 `apps`/`default` 리소스의 fallback) | 등록됨 |
| FastAPI | `apps/fastapi.yaml` | `fastapi` | `apps` | 계약 확정, 활성화 PR에서 등록 예정 |

FastAPI 외 서비스는 준비 상태와 namespace를 각각 확인한 뒤 서비스별 독립 PR로
`apps/<service>.yaml`과 `<service>/`를 추가합니다. 아직 확인되지 않은 서비스 이름,
경로와 namespace는 이 문서에서 임의로 예약하지 않습니다.

## 정적 검증 결과 (2026-08-17)

현재 브랜치의 `apps/*.yaml`과 각 source path, `INFRA/platform`의 root Application 및
기반 리소스 소유 범위를 비교했습니다. 실제 ArgoCD sync와 클러스터 조회는 수행하지
않았습니다.

| 검사 항목 | 결과 |
| --- | --- |
| Application 수 | 1개: `rabbitmq` |
| source path | `rabbitmq/`가 존재하며 `kustomization.yaml` 포함 |
| repoURL | Git remote `origin`과 동일한 `https://github.com/AIVLE-SELLoN/sellon-gitops.git` |
| targetRevision | child는 `main`, platform 변수 기본값도 `main` |
| 렌더링 | `kubectl kustomize rabbitmq` 성공, 17개 리소스 |
| 중복 identity | 렌더링 결과의 `(kind, namespace, name)` 중복 0건 |
| Secret 평문 | 이 Application 경로에 `Secret` 매니페스트 없음 |

RabbitMQ Application의 `destination.namespace`는 `default`이지만, 이 값은 namespace가
생략된 리소스의 기본 목적지일 뿐 명시된 namespace를 덮어쓰지 않습니다. 렌더링 결과는
`default` 13개와 `apps` 4개(User/Permission)입니다. `apps` namespace는 Application이
만들지 않으며 `INFRA/platform`이 먼저 생성해야 합니다.

현재 작업 브랜치는 `feat/gitops-common-bootstrap`이지만 Application은 `main`을
추적합니다. 따라서 이 브랜치의 변경은 `main`에 머지되기 전에는 ArgoCD에 반영되지
않습니다.

### 서비스 및 기반 준비 상태

준비되지 않은 서비스에는 Application 이름이나 path를 미리 할당하지 않습니다.

| 대상 | GitOps Application 대상 여부 | 준비 상태 |
| --- | --- | --- |
| RabbitMQ | 대상 | Application과 매니페스트 준비됨. platform 오퍼레이터·`apps` namespace·저장소 인증 확인 필요 |
| FastAPI | 대상 예정 | Application과 `fastapi/` 경로 모두 없음. 활성화 PR 전까지 미등록 유지 |
| Spring 백엔드 | 후보 | `apps` 배포 계약만 확인됨. 매니페스트와 Application은 준비되지 않음 |
| ChromaDB | 현재 대상 아님 | `INFRA/platform/k8s_chromadb.tf`가 Terraform으로 관리하므로 Application을 추가하지 않음 |
| ArgoCD, cert-manager, ESO, RabbitMQ 오퍼레이터 | 대상 아님 | `INFRA/platform`의 bootstrap 리소스로 유지 |
| RDS, ElastiCache 등 AWS data 리소스 | 대상 아님 | Terraform `data` 스택 소유 |

### 리소스 소유권과 prune 위험

- root Application은 `INFRA/platform`, child Application과 RabbitMQ CR은 이 저장소가
  소유하므로 실행 리소스의 직접 중복 소유는 확인되지 않았습니다.
- `INFRA/gitops/`에도 동일한 RabbitMQ Application과 매니페스트 복사본이 추적되고
  있습니다. Terraform이 그 디렉터리를 직접 적용하지는 않으므로 현재 즉시 이중 생성되지는
  않지만, root Application이 어느 저장소를 보느냐에 따라 정본이 갈릴 수 있습니다.
  활성화 전 `gitops_repo_url`이 이 저장소만 가리키는지 확인하고, `INFRA/gitops/`의
  폐기·보관 방침은 별도 PR로 정리해야 합니다.
- root와 RabbitMQ Application 모두 `prune: true`이고 child에는 resources finalizer가
  있습니다. `apps/rabbitmq.yaml`을 삭제하거나 이름을 바꾸면 child Application 삭제와
  관리 리소스 연쇄 삭제가 시작될 수 있습니다.
- `RabbitmqCluster`와 Exchange에는 `Prune=false,Delete=false`, Vhost와 Queue에는
  `deletionPolicy: retain`이 있어 핵심 데이터 경로가 보호됩니다.
- Binding, User, Permission은 의도적으로 prune 대상입니다. 특히 User가 삭제되면
  오퍼레이터 생성 자격증명 Secret과 애플리케이션 접속에 영향이 있으므로 이름 변경과
  제거는 회전 절차 없이 수행하지 않습니다.
- Namespace와 PVC를 이 Application이 직접 선언하지 않으므로 해당 리소스의 직접 중복
  소유는 없습니다. RabbitMQ 오퍼레이터가 생성하는 StatefulSet·PVC·Service·Secret은
  RabbitmqCluster/User CR의 하위 리소스이므로 CR 삭제 경로를 통해 간접 영향을 받습니다.

### 활성화 전 차단 항목

- private 저장소의 ArgoCD GitHub App 인증을 준비하고 repository 연결 성공 확인
- platform 실제 `gitops_repo_url`이 이 저장소 URL이고 `gitops_repo_branch`가 `main`인지 확인
- RabbitMQ 오퍼레이터가 준비되고 Terraform 소유 `apps` namespace가 존재하는지 확인
- `INFRA/gitops/` 복사본이 별도 Application에서 참조되지 않는지 확인하고 정본을 이 저장소로 고정
- 현재 브랜치 변경을 `main`에 머지한 뒤에만 활성화

## 서비스 등록 절차

1. `<service>/`에 독립 렌더링 가능한 매니페스트를 준비합니다.
2. Secret 평문, namespace 소유권, PVC 등 상태 보존 리소스와 prune 영향을 검토합니다.
3. 활성화가 가능한 시점에 `apps/<service>.yaml`을 별도 PR로 추가합니다.
4. `repoURL`, `targetRevision`, `path`, destination namespace가 root 및 서비스 계약과
   일치하는지 정적으로 검증합니다.
5. 실제 sync는 기반 리소스와 저장소 인증이 준비된 뒤 별도 배포 단계에서 수행합니다.

## 미결 입력값

- `INFRA/platform`의 실제 `gitops_repo_url`, `gitops_repo_branch` 입력 및 apply 여부
- private 저장소를 읽을 ArgoCD 인증정보의 원본, 주입 주체와 갱신 절차
- FastAPI 외 추가 등록 서비스의 이름, 매니페스트 경로와 destination namespace

## 안전 규칙

- 비공개 저장소 토큰, SSH private key, kubeconfig와 Secret 평문을 커밋하지 않습니다.
- 준비되지 않은 서비스의 Application을 미리 추가하지 않습니다.
- `automated.prune`을 사용하는 Application은 PVC, Queue, Namespace 등 상태 보존이
  필요한 리소스의 삭제 경로를 먼저 검토합니다. 기본값을 사용하기 어렵다면 해당
  서비스 PR에서 예외와 보호 방법을 함께 문서화합니다.
- Application은 namespace를 자동 생성하지 않습니다. 대상 namespace는 Terraform 등
  합의된 주체가 먼저 생성해야 합니다.
- 실제 ArgoCD sync와 `kubectl apply/delete`는 별도의 활성화 단계에서 수행합니다.
