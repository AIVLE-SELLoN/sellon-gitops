# ArgoCD private 저장소 인증

## 현재 상태

- ArgoCD 설치와 root Application 정의는 `INFRA/platform`이 담당합니다.
- root Application은 `gitops_repo_url`이 비어 있지 않을 때만 생성되며, 저장소의
  `apps/` 경로를 읽습니다.
- `sellon-gitops`는 private 저장소이지만, 현재 코드에는 ArgoCD repository Secret,
  credential template(`repo-creds`), ExternalSecret 또는 GitHub 인증정보를 주입하는
  구성이 없습니다.
- 따라서 현재 상태에서 `gitops_repo_url`만 입력해 root Application을 만들면,
  ArgoCD가 저장소를 읽지 못해 동기화가 시작되지 않을 수 있습니다.

실제 클러스터에 repository 인증이 별도로 등록됐는지와 root Application의 현재
생성 여부는 이 저장소의 정적 코드만으로 확인할 수 없습니다.

## 확정된 방식과 미결 항목

다음 방향은 확정됐지만 아직 `INFRA/platform` 코드에는 구현되지 않았습니다.

- 인증 방식: 개인 PAT가 아닌 GitHub App
- 저장소 범위: `AIVLE-SELLoN/sellon-gitops` 하나만 선택
- 권한: repository Contents read-only 최소 권한
- 인증정보 원본: AWS Secrets Manager의 `sellon/argocd/github-app`
- 주입 방식: platform의 ESO가 `argocd` namespace에 ArgoCD repository Secret 생성
- 적용 순서: 인증을 먼저 준비·확인한 뒤 root Application 생성

GitHub 조직 관리 역할이 App 생성과 저장소 설치를 담당하고, 인프라 관리 역할이
Secrets Manager 원본·ESO 주입·갱신 절차를 관리합니다. 실제 담당자, 자격증명 갱신
주기와 실행 절차는 아직 정해야 합니다. `INFRA/platform`에 실제 입력할 저장소 URL과
target revision 및 현재 apply 상태도 별도 확인이 필요합니다.

## 안전한 최초 연결 순서

1. GitHub App을 만들고 `sellon-gitops` 저장소에 Contents read-only로 설치합니다.
2. App ID, Installation ID와 private key를 `sellon/argocd/github-app`에 별도로
   등록합니다. 실제 값은 Git, Terraform 변수·state 또는 문서에 기록하지 않습니다.
3. `gitops_repo_url=""` 상태에서 platform의 ESO 주입 구성을 먼저 적용합니다.
4. ArgoCD의 repository 연결 상태가 성공이고 `main` 브랜치를 읽을 수 있는지
   확인합니다.
5. 그다음 `INFRA/platform`의 `gitops_repo_url`과 `gitops_repo_branch`를 실제 값으로
   입력하고 Terraform을 적용해 root Application을 생성합니다.
6. root Application의 소스 URL·revision과 동기화 상태를 확인합니다.

이 순서는 향후 운영 절차이며, 이 문서 작업에서는 repository Secret 생성,
Terraform apply, ArgoCD sync 또는 서비스 Application 활성화를 수행하지 않습니다.

## 선언형 주입 구현 계약

ArgoCD repository 인증은 `argocd` namespace의 Kubernetes Secret으로 저장되며,
`argocd.argoproj.io/secret-type: repository` 라벨과 저장소 URL·인증 방식에 맞는 필드를
사용합니다. 다만 private 저장소를 읽기 위한 인증정보를 그 private 저장소 자신에게만
두면 최초 동기화를 시작할 수 없으므로, bootstrap 단계에서 먼저 주입할 경로가
필요합니다.

구현 시 소유 범위는 다음과 같이 둡니다.

- `INFRA/platform`이 `sellon/argocd/github-app` Secret 틀과 해당 ARN의 ESO 읽기 권한을 관리
- 실제 GitHub App 값은 Terraform 밖에서 Secrets Manager에 등록
- platform bootstrap 구성이 `argocd` namespace의 SecretStore와 ExternalSecret을 관리
- ExternalSecret이 ArgoCD repository Secret을 생성하고 root Application보다 먼저 준비

이 항목들은 현재 구현되어 있지 않으며, 이 저장소에는 평문 Secret 매니페스트를
추가하지 않습니다.

## 갱신 절차

1. 원본 시스템에서 새 자격증명을 발급합니다.
2. ArgoCD의 repository 인증정보를 새 값으로 교체합니다.
3. repository 연결과 root Application 새로고침이 성공하는지 확인합니다.
4. 성공을 확인한 뒤 이전 자격증명을 폐기합니다.

구체적인 명령과 자동화 방식은 실제 담당자와 갱신 주기가 결정된 뒤 추가합니다.
갱신 과정에서 Secret 값을 로그나 명령 이력에 출력하지 않습니다.

## 배포 전 확인 목록

- [x] GitHub App과 저장소 한정 Contents read-only 권한 확정
- [x] Secrets Manager 원본과 platform ESO 주입 방식 확정
- [ ] GitHub App 생성·설치 및 Secret 갱신 담당자 확정
- [ ] platform bootstrap 인증 코드 구현
- [ ] ArgoCD repository 연결 성공 확인
- [ ] platform의 실제 `gitops_repo_url`과 `gitops_repo_branch` 확인
- [ ] 인증 준비 후 root Application 생성 및 상태 확인

## 참고 문서

- [Argo CD - Private Repositories](https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/)
- [Argo CD - Declarative Setup](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/)
