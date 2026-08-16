# ArgoCD Application 규칙

`INFRA/platform`이 생성하는 root Application은 이 디렉터리를 재귀적으로 읽습니다.
여기에는 서비스별 ArgoCD `Application`만 두고 Deployment, Service, Secret 같은
워크로드 리소스는 두지 않습니다.

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

## 안전 규칙

- 비공개 저장소 토큰, SSH private key, kubeconfig와 Secret 평문을 커밋하지 않습니다.
- 준비되지 않은 서비스의 Application을 미리 추가하지 않습니다.
- `automated.prune`을 사용하는 Application은 PVC, Queue, Namespace 등 상태 보존이
  필요한 리소스의 삭제 경로를 먼저 검토합니다.
- 실제 ArgoCD sync와 `kubectl apply/delete`는 별도의 활성화 단계에서 수행합니다.
