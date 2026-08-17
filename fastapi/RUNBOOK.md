# FastAPI 운영 · Rollback Runbook

## 범위와 전제

이 문서는 `fastapi-ai-node` 서비스의 네 워크로드 — Web(Deployment),
Consumer(Deployment), Classification Worker(CronJob), Daily
Batch(CronJob) — 의 운영·장애대응·rollback 절차를 다룬다.

이 문서가 인용하는 스케줄/포트/리소스 이름 등은 `fastapi/core`,
`fastapi/batch`, `fastapi/policies`, `fastapi/rbac`에 걸쳐 이미 확정된
계약이다. 이 문서를 작성한 브랜치(`feat/fastapi-network-operations`)에는
`fastapi/core`·`fastapi/batch`가 없어(별도 브랜치 소유) 이 문서에서 그
파일을 직접 렌더링해 대조할 수는 없었다 — 이번 세션에서 그 브랜치들에
직접 적용한 값을 그대로 인용한 것이며, 병합 후에는 실제 매니페스트와
어긋나지 않는지 다시 확인해야 한다.

이 문서 자체는 절차 기술 문서이며, 여기 적힌 어떤 명령도 이 세션에서
실행하지 않았다 — `kubectl apply`/`ArgoCD sync`/`terraform apply`는
수행하지 않는다는 저장소 규칙(AGENTS.md)이 이 문서 작성에도 그대로
적용된다.

## 1. Web

- **Health**: `GET /health`, `containerPort: 8080`. readiness/liveness
  probe가 동일 엔드포인트를 사용한다(startup probe 없음).
- **확인 절차**:
  ```
  kubectl -n apps get pods -l app.kubernetes.io/component=web
  kubectl -n apps logs -l app.kubernetes.io/component=web --tail=200
  kubectl -n apps port-forward svc/fastapi-ai-node-web 8080:8080
  curl -sf localhost:8080/health
  ```
- readiness 실패 시 Service 엔드포인트에서 자동 제외된다. `RollingUpdate
  maxUnavailable: 0 / maxSurge: 1`이지만 `replicas: 1`이므로, 배포 중
  "무중단"은 새 Pod가 Ready 될 때까지 이전 Pod가 잠깐 같이 떠 있는
  것뿐이고 실질적인 동시 가동 구간은 매우 짧다 — 완전한 무중단을 보장하는
  설정은 아니라는 점을 장애대응 시 감안한다.

## 2. Consumer

- **Manual ACK / prefetch 10**: 애플리케이션 코드 계약이며 이 매니페스트로
  강제되지 않는다. 큐 적체 시 prefetch 10이 처리량 병목이 될 수 있다는
  것을 인지하고, 적체가 계속되면 애플리케이션 로그로 ACK 지연/미처리
  원인을 먼저 확인한다(레플리카를 늘려 우회하지 않는다 — prefetch/ACK
  계약은 코드 쪽 설계이지 스케일 문제가 아닐 수 있다).
- **종료 코드 계약**: `terminationGracePeriodSeconds: 30`. SIGTERM 수신 시
  새 메시지 소비를 멈추고, in-flight 메시지를 ack/nack 처리한 뒤 채널·
  커넥션을 정리하고 종료해야 한다. Consumer는 CronJob이 아니라
  Deployment이므로, 컨테이너가 0이 아닌 종료 코드로 끝나면 kubelet이
  자동 재시작하고 반복되면 `CrashLoopBackOff`가 된다.
- **확인 절차**:
  ```
  kubectl -n apps get pods -l app.kubernetes.io/component=consumer
  kubectl -n apps describe pod <pod>        # Last State/Exit Code/Reason
  kubectl -n apps logs --previous <pod>
  ```

## 3. Classification Worker (CronJob)

- **schedule**: `0 2 * * *`, `timeZone: Asia/Seoul`(02:00 KST),
  `activeDeadlineSeconds: 1800`, `concurrencyPolicy: Forbid`.
- **확인**:
  ```
  kubectl -n apps get cronjob fastapi-ai-node-classification-worker
  kubectl -n apps get jobs -l app.kubernetes.io/component=classification-worker
  ```
- 1800초를 넘기면 Job이 `DeadlineExceeded`로 강제 종료된다 — 원인 조사는
  해당 Job의 Pod 로그(LLM 응답 지연, raw DB 락/대기, 배치 대상 건수 급증
  등)부터 확인한다.
- `concurrencyPolicy: Forbid`이므로 이전 실행이 끝나지 않으면 다음
  스케줄은 아예 Job을 만들지 않고 건너뛴다(에러가 아니라 "스킵"). 놓친
  실행 여부는 `status.lastScheduleTime`과 실제 스케줄을 비교해 확인한다:
  ```
  kubectl -n apps get cronjob fastapi-ai-node-classification-worker \
    -o jsonpath='{.status.lastScheduleTime}'
  ```

## 4. Daily Batch (CronJob)

- **schedule**: `30 2 * * *`, `timeZone: Asia/Seoul`(02:30 KST),
  `activeDeadlineSeconds: 3600`, `concurrencyPolicy: Forbid`.
- **PVC 확인**: 상태 저장용 PVC(`fastapi-ai-node-daily-batch-state`, gp3,
  1Gi, RWO, `/app/data/batch_state`에 마운트)가 `Bound` 상태인지, 용량이
  임박하지 않았는지 먼저 확인한다:
  ```
  kubectl -n apps get pvc fastapi-ai-node-daily-batch-state
  kubectl -n apps describe pvc fastapi-ai-node-daily-batch-state
  ```
  `describe`의 Events에서 attach/mount 실패가 보이면 `concurrencyPolicy:
  Forbid` + RWO 조합의 특성을 먼저 의심한다 — 이전 실행의 Pod가
  Terminating 상태로 오래 붙어 있으면 볼륨이 아직 detach되지 않아 다음
  실행이 attach 대기로 지연/실패할 수 있다.
- 3600초를 넘기면 `DeadlineExceeded`로 종료된다. 원인 조사 절차는
  Classification Worker와 동일하되, S3 업로드·raw DB 조회 구간도 함께
  본다.

## 5. 이미지 rollback (`main-<shortSHA>`)

- **GitOps 우선**: 이 저장소가 정본이므로, 정상 롤백은 매니페스트의
  `image:` 태그를 이전 known-good `main-<shortSHA>`로 되돌리는 커밋을
  만들어 리뷰 후 merge하고, ArgoCD sync로 반영하는 방식으로 한다.
  `kubectl set image`처럼 클러스터를 직접 바꾸는 방법은 GitOps 상태와
  어긋나는 drift를 만든다 — auto-sync가 켜져 있으면 다음 sync에서 그
  즉흥 변경이 도로 원래(문제였던) 이미지로 되돌아갈 수 있고, 꺼져 있으면
  drift가 계속 남아 다음 사람이 실제 배포 상태를 오판하게 만든다.
- **절차**:
  1. git 이력(커밋/PR)에서 이전 known-good `shortSHA`를 확인한다.
  2. `fastapi/core`(Web/Consumer)·`fastapi/batch`(Classification
     Worker/Daily Batch)의 해당 `image:` 필드를 그 shortSHA로 되돌리는
     커밋을 만든다.
  3. 리뷰 후 merge하고, ArgoCD sync를 확인한다(이 세션에서는 sync를
     수행하지 않는다).
- **전제 조건 — 아직 닫히지 않음**: Docker Hub 네임스페이스가
  `<DOCKERHUB_NAMESPACE>` placeholder로 남아 있고, Classification
  Worker가 쓰는 `scripts/`-포함 이미지는 Dockerfile PR merge 여부와
  linux/amd64 이미지 push 여부가 이 세션에서 확인되지 않았다(`fastapi
  /batch/INPUTS.md` "PR 6 blockers" 참고 — 이 브랜치에는 그 파일이 없다).
  즉 이 전제가 닫히기 전에는 "롤백할 이전 known-good 이미지" 자체가 아직
  없을 수 있다 — 그 상태에서 rollback 절차를 시도하면 무엇으로도 되돌릴
  대상이 없다는 점을 먼저 확인한다.

## 6. CronJob suspend

- **필드**: `spec.suspend: true`. 이 값도 GitOps 저장소가 정본이므로,
  `kubectl patch`로 직접 켜고 끄면 ArgoCD auto-sync 정책에 따라 되돌아갈
  수 있다 — 이 저장소가 실제로 auto-sync를 쓰는지, prune/self-heal이
  켜져 있는지는 `apps/` Application 정의를 확인해야 하며, 이 문서
  작성 시점에는 `apps/fastapi.yaml`이 아직 존재하지 않아 확인하지
  못했다(입력값 항목 참고).
- **사용 시점 예시**: 확정되지 않은 전제(raw PostgreSQL DSN, Docker Hub
  네임스페이스, 이미지 push 여부 등)가 남아 있는 동안 오발동을 막기 위해
  `suspend: true`를 유지하는 것도 하나의 선택지다 — 다만 이는 운영
  판단이므로 이 문서는 "가능한 절차"만 기술하고, 지금 값을 바꾸라고
  지시하지 않는다.
- **확인**:
  ```
  kubectl -n apps get cronjob <name> -o jsonpath='{.spec.suspend}'
  ```

## 7. 최초 Chroma 시딩 — 클러스터 Job 금지, 개발 머신 port-forward만

이 저장소에는 시딩용 cluster Job/CronJob이 없다 — 의도적인 결정이며,
`fastapi/batch/INPUTS.md`(형제 브랜치)에 이미 기록된 정책과 동일하다.

- **절차**:
  ```
  kubectl -n default port-forward svc/chromadb 8000:8000
  ```
  ChromaDB는 `apps`가 아니라 `default` 네임스페이스에 있다(`fastapi
  /policies`의 네임스페이스 경계 문서 참고). 이후 로컬 머신에서 AI
  저장소의 시딩 스크립트를 개발자 자격증명으로 포워딩된 포트를 대상으로
  실행한다 — 클러스터 Job으로 실행하거나 CronJob의 Secret 기반 운영
  자격증명을 사용하지 않는다.
- **`--reset` 금지**: 이 절차·스크립트·CI 어디에도 `--reset`을 자동화하지
  않는다. 클러스터 Seed Job이 없어 이 초기화를 견제할 별도 장치가 없으므로,
  개발자가 port-forward로 직접 라이브 데이터셋을 초기화하는 사고를 막는
  책임이 이 절차를 따르는 사람에게 있다.

## 검증 결과 (이 세션에서 한 것)

- 이 문서는 매니페스트가 아니라 운영 문서이므로 `kubectl kustomize` 렌더링
  대상이 아니다. 대신 문서에 인용한 값(스케줄, 포트, activeDeadlineSeconds,
  PVC 이름/사양, prefetch/manual ACK, `--reset` 금지)을 이번 세션에서
  이미 확정한 형제 브랜치의 값과 대조해 일관성을 확인했다.
- `git status --short`로 이 파일 하나만 추가됐고 범위 밖 파일은 건드리지
  않았음을 확인했다.

## 남은 입력값

| 항목 | 상태 | 담당 |
| --- | --- | --- |
| Docker Hub 네임스페이스 확정 (rollback 대상 이미지 경로 완성에 필요) | TBD | Backend/Infra |
| Classification Worker용 `scripts/` 이미지의 Dockerfile PR merge + linux/amd64 push 여부 | 이 세션에서 확인 불가 | AI팀/Backend |
| raw PostgreSQL DSN/계정 계약 (Classification Worker·Daily Batch가 실제로 정상 기동하는지에 영향) | TBD | AI팀 |
| `apps/fastapi.yaml`의 ArgoCD sync 정책(auto-sync, self-heal, prune 여부) — `suspend`/이미지 롤백 시 kubectl 직접 조작이 drift로 남는지 되돌아가는지를 결정 | 아직 `apps/fastapi.yaml`이 없어 확인 불가 | Backend/Infra |
