# NetworkPolicy 렌더링 검증 및 장애 테스트 계획 (PR 6)

이 문서는 `fastapi/policies`의 5개 `NetworkPolicy`가 (1) 정적으로
의도대로 렌더링되는지, (2) 실제 클러스터에 적용된 뒤 트래픽이 의도대로
허용/차단되는지 확인하기 위한 계획을 담는다. 이 세션에서는 (1)만
수행했고, (2)는 PR 6이 실제로 적용될 때 실행할 테스트 계획만
문서화한다 — `kubectl apply`/ArgoCD sync는 이 세션에서 수행하지 않았다.

## 1. 이번 세션에서 수행한 정적 검증

`kubectl kustomize fastapi/policies`로 5개 리소스를 렌더링해 다음을
확인했다(전부 통과):

| 확인 항목 | 결과 |
| --- | --- |
| DNS 허용 | `fastapi-ai-node-common-dns`가 `app: fastapi-ai-node` 전체(4개 워크로드 공통)에 kube-system/kube-dns 53(UDP/TCP)만 허용 |
| RabbitMQ 5672 허용 범위 | `fastapi-ai-node-consumer`와 `-daily-batch`에만 존재. 둘 다 RabbitMQ 발행/소비 주체다(Daily 는 `publish_anomaly_analyzed`·`publish_guideline_generated` 발행). Web/classification-worker 렌더링 결과에는 5672가 없음 |
| ChromaDB 8000 허용 범위 | `fastapi-ai-node-web`/`-consumer`/`-daily-batch`에만 존재. classification-worker 렌더링 결과에는 8000이 없음 |
| 외부 HTTPS 443 | 4개 워크로드 전부에 `ipBlock: 0.0.0.0/0` (RFC1918 제외) + port 443만 존재 |
| Redis 6379 | `grep -rn "port: 6379" fastapi/policies` 결과 0건 — 어떤 정책에도 6379가 없음(문서 내 설명 주석에만 문자열로 언급) |
| raw RDS PostgreSQL 5432 허용 범위 | `fastapi-ai-node-web`/`-classification-worker`/`-daily-batch`에만 존재, ipBlock은 `10.0.48.0/24`+`10.0.49.0/24` 두 개(병합 안 됨). `fastapi-ai-node-consumer` 렌더링 결과에는 5432가 없음 |
| "그 외 전부 차단" | 5개 정책 모두 `policyTypes: [Egress]`이고 각 워크로드 Pod는 최소 1개(자기 컴포넌트 정책) + common-dns 정책의 대상이 되므로, 명시되지 않은 egress는 Kubernetes NetworkPolicy 기본 동작(egress 정책이 하나라도 걸린 Pod는 명시된 규칙 외 전부 거부)에 따라 차단됨 |

## 2. PR 6 적용 후 실행할 테스트 — 사전 조건

- `fastapi/core`(Web/Consumer), `fastapi/batch`(classification-worker/
  daily-batch), `fastapi/rbac`, `fastapi/policies`가 모두 병합되어 실제
  Pod가 해당 라벨(`app.kubernetes.io/component=...`)로 떠 있어야 한다.
- 실제 워크로드 Pod가 아직 없다면, 동일한 라벨만 붙인 임시 테스트 Pod로
  대체 가능하다(NetworkPolicy는 podSelector 라벨만 보고 적용되므로):
  ```
  kubectl run netpol-test-web -n apps --rm -it \
    --image=nicolaka/netshoot \
    --labels="app.kubernetes.io/name=fastapi-ai-node,app.kubernetes.io/component=web" \
    -- sh
  ```
  workload마다 `app.kubernetes.io/component` 값만 바꿔 동일하게 만든다
  (`consumer`, `classification-worker`, `daily-batch`).
- 판정 기준: 허용 케이스는 수 초 내 TCP 연결(3-way handshake) 또는
  애플리케이션 응답 성공. 차단 케이스는 지정한 타임아웃(예: 5초) 내
  연결이 열리지 않아야 한다 — NetworkPolicy egress 차단은 보통 SYN이
  드롭되어 "timeout"으로 나타난다. 즉시 "Connection refused"가 나오면
  방화벽이 아니라 상대가 포트를 닫고 있어서일 수 있으므로 원인을 다시
  구분해야 한다.

## 3. 성공 테스트 케이스 (연결 성공해야 함)

| # | 워크로드 | 대상 | 포트 | 명령 예시 |
| --- | --- | --- | --- | --- |
| S1 | 전체(4개 워크로드 공통) | kube-dns | 53 | `nslookup kubernetes.default` |
| S2 | Web | ChromaDB(`default`) | 8000 | `nc -zv -w3 chromadb.default.svc.cluster.local 8000` |
| S3 | Web | 외부 LLM API | 443 | `curl -sS --max-time 5 -o /dev/null -w '%{http_code}\n' https://<LLM_ENDPOINT_TBD>` |
| S4 | Web | raw RDS PostgreSQL | 5432 | `nc -zv -w3 <RDS_ENDPOINT_TBD> 5432` |
| S5 | Consumer | RabbitMQ(`default`) | 5672 | `nc -zv -w3 rabbitmq.default.svc.cluster.local 5672` |
| S6 | Consumer | ChromaDB(`default`) | 8000 | `nc -zv -w3 chromadb.default.svc.cluster.local 8000` |
| S7 | Consumer | 외부 LLM API | 443 | S3와 동일 명령 |
| S8 | classification-worker | 외부 LLM API | 443 | S3와 동일 명령 |
| S9 | classification-worker | raw RDS PostgreSQL | 5432 | S4와 동일 명령 |
| S10 | daily-batch | ChromaDB(`default`) | 8000 | S2와 동일 명령 |
| S11 | daily-batch | RabbitMQ(`default`) | 5672 | S5와 동일 명령. Daily 는 이상탐지·가이드라인 이벤트를 발행하므로 반드시 성공해야 함 |
| S12 | daily-batch | 외부 LLM API + S3 | 443 | S3와 동일 명령(LLM/S3 각각) |
| S13 | daily-batch | raw RDS PostgreSQL | 5432 | S4와 동일 명령 |
| S14 | daily-batch | spring-backend(`apps`, `app: spring-backend`) | 8080 | `nc -zv -w3 <spring-backend-pod-ip> 8080` |

## 4. 실패(차단) 테스트 케이스 — 연결이 반드시 막혀야 함

| # | 워크로드 | 대상 | 포트 | 차단 근거 |
| --- | --- | --- | --- | --- |
| F1 | Web | RabbitMQ(`default`) | 5672 | Web은 RabbitMQ 접근 권한 없음(Consumer 전용) |
| F2 | Consumer | raw RDS PostgreSQL | 5432 | 이번 정책에서 Consumer는 raw DB egress 대상에서 명시적으로 제외됨 |
| F3 | classification-worker | ChromaDB(`default`) | 8000 | classification-worker는 Chroma egress 없음(다른 PR에서 이미 확정된 결정) |
| F4 | 4개 워크로드 전체 | ElastiCache/Redis | 6379 | Redis 미사용, 어떤 정책도 6379를 열지 않음 |
| F5 | Web/Consumer/classification-worker | spring-backend(`apps`) | 8080 | spring-backend 접근은 daily-batch만 허용 |
| F6 | classification-worker | RabbitMQ(`default`) | 5672 | classification-worker는 RabbitMQ egress 없음 |
| F7 | 아무 워크로드 | 임의 외부 호스트 | 80 (443 아님) | 443만 허용, 80을 포함한 다른 포트는 전부 차단 |
| F8 | 아무 워크로드 | RFC1918 사설 대역(예: 클러스터 내 다른 Pod/Service) 상의 443 | 443 | 외부 HTTPS 규칙이 `except: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16`으로 사설 대역을 제외했으므로, "외부 443 허용"이 내부 443 서비스로 새지 않아야 함 |
| F9 | classification-worker | 외부 LLM API 이외의 그 무엇이든 (S3 등) | 443 | classification-worker에 S3 egress 권한이 부여된 적 없음(이번 정책에서 S3는 daily-batch에만 근거가 있음) — 443을 열어둔 것 자체가 "무엇이든 허용"을 뜻하지 않는다는 점을 재확인하는 케이스이며, 도메인 단위 구분은 NetworkPolicy가 못 하므로 실제로는 통과(허용)한다 — 이 한계는 F-케이스가 아니라 알려진 제약으로 별도 기록(5절 참고) |

F9은 사실 "차단되지 않는" 케이스로 기록한다 — NetworkPolicy가 도메인을
구분하지 못하는 한계 때문에 발생하는, 설계상 알려진 갭이지 이번
정책의 결함이 아니다. 자세한 내용은 `fastapi/policies/INPUTS.md`
"External domain limitation"을 참고한다.

## 5. 알려진 한계 (테스트 결과를 해석할 때 참고)

- 외부 HTTPS 443 규칙은 "LLM만" 또는 "S3만"으로 좁혀지지 않는다. 포트
  443의 어떤 외부 호스트로도 나갈 수 있다(RFC1918 제외). F9이 그 갭을
  보여주는 케이스다.
- RabbitMQ/ChromaDB 규칙은 특정 Pod가 아니라 `default` 네임스페이스 +
  포트로만 좁혀져 있다(Pod 라벨이 플랫폼 소유라 이 저장소에서 확정하지
  못함). 따라서 이론적으로는 `default` 네임스페이스의 같은 포트를 쓰는
  다른 서비스로도 나갈 수 있다 — 지금 시점에 `default`에 그런 서비스가
  없다면 실질적 위험은 없지만, 나중에 추가되면 재검토가 필요하다.

## 6. 원복 절차 (문제 발생 시)

1. **즉시 완화**: 문제되는 `NetworkPolicy`를 클러스터에서 직접
   제거한다(`kubectl -n apps delete networkpolicy <name>`). 이는 GitOps
   상태와 어긋나는 임시 조치이므로 오래 유지하지 않는다.
2. **저장소 원복(정본)**: PR 6에서 추가된 `fastapi/policies/*.yaml` 커밋을
   되돌리는 revert PR을 만들어 리뷰 후 병합한다 — `fastapi/RUNBOOK.md`
   5장에 이미 기록한 "GitOps 우선" 원칙과 동일하게, 실제 정본은 이
   저장소이지 클러스터의 즉흥 상태가 아니다.
3. **ArgoCD sync 확인**: revert가 병합된 뒤 sync가 실제로 반영됐는지
   확인한다(이 세션에서는 sync를 수행하지 않는다).
4. **재확인**: `kubectl get networkpolicy -n apps`로 문제였던 정책이
   제거됐는지 확인하고, 원래 있던 (제한 없는) egress 상태로 돌아왔는지
   1절의 테스트 케이스 중 관련된 항목만 다시 돌려 확인한다.
5. **부분 원복도 가능**: 5개 파일 중 문제가 된 워크로드 하나만
   revert하는 것도 가능하다 — 파일이 워크로드별로 이미 분리되어 있어
   전체를 되돌릴 필요는 없다.

## 7. 남은 입력값 (실제 테스트 실행 전 채워야 함)

| 항목 | 필요 이유 | 담당 |
| --- | --- | --- |
| LLM API 실제 엔드포인트(도메인) | S3/S7/S8/S12 테스트 명령의 `<LLM_ENDPOINT_TBD>` | AI팀/Backend |
| raw RDS PostgreSQL 실제 엔드포인트 | S4/S9/S13 테스트 명령의 `<RDS_ENDPOINT_TBD>` | Infra |
| spring-backend 실제 Pod (아직 이 저장소에 매니페스트 없음) | S14는 spring-backend가 `apps`에 실제로 떠야 실행 가능 | Backend(Spring팀) |
| RabbitMQ/ChromaDB 실제 Pod 라벨 | F-케이스 해석 시 "네임스페이스 전체 허용" 범위를 더 좁게 확인하려면 필요(5절 한계 참고) | 플랫폼팀 |
