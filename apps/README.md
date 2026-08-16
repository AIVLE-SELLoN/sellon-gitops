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
| FastAPI | `apps/fastapi.yaml` | `fastapi` | `apps` | 계약 확정, 활성화 PR에서 등록 예정 |

FastAPI 외 서비스는 준비 상태와 namespace를 각각 확인한 뒤 서비스별 독립 PR로
`apps/<service>.yaml`과 `<service>/`를 추가합니다. 아직 확인되지 않은 서비스 이름,
경로와 namespace는 이 문서에서 임의로 예약하지 않습니다.

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
