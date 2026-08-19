# Index

Qket 프로젝트 팀 위키의 전체 페이지 목록. 새 페이지를 추가하면 여기 등록한다.

## 산출물 (제출용 문서) — 표준 6개 카테고리 밖의 예외
- [[기획서]] — 제출용 프로젝트 기획서. 위키 내용을 근거로 작성, Obsidian PDF 내보내기 전제로 포맷팅됨(다이어그램 포함 단일 문서)

## onboarding/
- [[getting-started]] — 신규 팀원이 제일 먼저 읽는 페이지, 체크리스트 포함

## conventions/ — "우리는 이렇게 코딩한다"
- [[comment-rules]] — 백엔드 메서드 주석 / 프론트 API 함수 주석 형식
- [[api-response-format]] — `ApiResponse`/`ErrorResponse` 규격
- [[mybatis-conventions]] — XML 위치, type-alias 등록, nullable FK 가드 함정
- [[api-client-usage]] — `apiFetch` 사용 규칙
- [[admin-grid-pattern]] — 관리자 그리드 페이지 dirty-tracking 패턴
- [[css-file-structure]] — `styles/*.css` 역할별 분리 구조
- [[reusable-ui-components]] — `components/ui/` 컴포넌트 목록과 용도

## architecture/ — "시스템이 이렇게 동작한다"
- [[backend-layer-and-package-structure]] — Controller→Service→Mapper 4단 구조, `com.exam.*` 패키지
- [[auth-and-authorization]] — 세션/Role/IP 추적
- [[frontend-folder-structure]] — `lib/api`, `lib/data/types` 구조
- [[dynamic-menu-system]] — PROGRAMS/ROLE_PROGRAMS 기반 메뉴, 화면 노출 vs API 보안 구분
- [[db-schema-conventions]] — 감사 컬럼 6종
- [[terraform-remote-state]] — S3 backend, infrastructure/data(구 platform/workload) 간 `terraform_remote_state` 참조 구조
- [[terraform-module-boundaries]] — vpc/subnet/security_group/ec2/eks/rds/redis/storage/ecr 모듈 경계
- [[terraform-platform-workload-split]] — infrastructure(공유)/data(workspace) 2-root 구조(구 platform/workload), env_config 패턴, nightly-off 상태를 방어하는 `try()` 패턴
- [[admin-ingress-shared-alb]] — Grafana+ArgoCD IP 허용목록 공유 ALB, IngressGroup이 `inbound-cidrs`/`healthcheck-path`를 적용하는 단위
- [[messaging-infrastructure]] — 이메일 발송(SQS+Lambda+SES) 구조, root 배치, Lambda 배포 방식
- [[keda-autoscaling]] — backend/frontend KEDA 오토스케일링 구조(엔진은 Infra/Terraform, ScaledObject는 CD/Helm), metrics-server 전제조건, ArgoCD `ignoreDifferences`로 replicas 충돌 방지, 파드 오토스케일링과 노드 오토스케일링은 별개라는 점
- [[cluster-autoscaler]] — 노드 레벨 오토스케일링(EKS 관리형 노드그룹 desired_size 자동 조절), Karpenter 대신 선택한 이유, ASG autodiscovery 태그가 이미 자동으로 붙어있다는 점, IRSA 전파 지연으로 최초 1회 크래시하는 게 정상이라는 점

## decisions/ — "왜 이렇게 하기로 했나" (ADR)
- [[README]] — 사용법/템플릿
- [[2026-08-06-ci-tool-github-actions]] — CI 도구로 GitHub Actions 선택 (Jenkins 대신)
- [[2026-08-06-terraform-module-restructure]] — Terraform 모듈 리소스타입별 재편 + platform/workload 2-root화
- [[2026-08-06-ecr-recreate-vs-import]] — 기존 ECR 저장소 import 대신 삭제 후 재생성
- [[2026-08-10-redis-session-queue-shared-instance-risk]] — 세션/대기열이 같은 Redis 인스턴스 공유 시 리스크(noisy neighbor, 메모리 압박), 페일오버로도 안 풀리는 부분, 물리 분리 vs 논리 DB 분리 비용 비교 (논의중 — 부하테스트 후 재검토, 2026-08-18 대용량 트래픽 분석에서도 다시 보류 확정)
- [[2026-08-10-cd-writeback-github-app]] — CD 레포 write-back 인증으로 GitHub App 선택(PAT 대신) — 장기 자격증명을 시크릿에 안 두려는 원칙, 개인 계정으로 잘못 생성됐던 시행착오 포함
- [[2026-08-11-monitoring-stack-design]] — 모니터링 스택 설계(kube-prometheus-stack + Loki + CloudWatch 연동), 로그/지표 영구 저장소를 `03_registry`에 두기로 한 이유, 1차/2차 범위 구분 (범위 축소해서 구현 완료 — Loki는 빠짐, 실제 구현 결과는 문서 하단 참고)
- [[2026-08-11-external-api-secrets-manager]] — OAuth 3사/Toss 키를 Secrets Manager 새 시크릿(`external_api`)+ESO로 관리, `ignore_changes`로 사람이 채운 값 보호
- [[2026-08-11-frontend-api-routing-alb-not-rewrites]] — `/api` 라우팅을 Next.js `rewrites()`(빌드 타임 고정, 실제 프로덕션 버그 발생)에서 ALB path 라우팅으로 이전
- [[2026-08-11-frontend-ci-toss-key-secrets-manager]] — 프론트 CI 빌드 시점 `NEXT_PUBLIC_TOSS_CLIENT_KEY`를 GitHub Secrets 대신 Secrets Manager에서 OIDC로 직접 fetch
- [[2026-08-11-vpn-access-control-paused]] — dev/Grafana/CD VPN 접근제어 논의(WireGuard+internal ALB 방향으로 기울었으나 팀 상의 필요해 보류, 결정 아님)
- [[2026-08-12-messaging-infra-placement]] — 이메일 발송 인프라를 새 root 대신 `04_data`+`03_registry`에 배치, Lambda를 CI 없이 로컬 zip으로 배포
- [[2026-08-12-sqs-lambda-ses-notification-pipeline]] — 예매확정/취소 알림 파이프라인으로 SQS→Lambda→SES 채택(SNS/EventBridge Scheduler 대신) 근거 — 팀원 공유 아키텍처 노트 기반, 레포 분리/`05_messaging` root 제안은 참고만 하고 안 따름
- [[2026-08-18-capacity-planning-large-traffic-readiness]] — 대용량 트래픽 대비 용량 분석(실측 기반): cluster-autoscaler/Karpenter 부재가 가장 큰 병목(KEDA가 파드를 늘려도 노드가 안 늘어남), RDS 커넥션 여유·버스터블 크레딧 리스크, Redis 단일노드 재확인 — **같은 날 cluster-autoscaler 설치 + RDS release 상향(db.t3.medium)까지 완료, Redis는 사용자가 보류 결정, 장시간 재측정은 미착수**
- [[2026-08-19-queue-scope-limited-to-booking-flow]] — 대기열이 "예매하기" 클릭 이후만 보호하고 홈/상세(부하테스트에서 제일 먼저 무너졌던 구간)는 사각지대라는 게 드러났는데, 지금 규모에선 대기열 범위 확장(대안 A) 대신 캐싱+헬스체크 분리(GitHub 이슈 3건)로 완화하기로 확정 — 실제 트래픽이 지금 규모를 크게 넘으면 재검토하기로 트리거를 남겨둠

## troubleshooting/ — 실제 겪은 버그
- [[null-field-partial-update-bug]] — nullable FK 부분 업데이트 버그 (백/프론트 양쪽)
- [[utf8mb4-encoding-bug]] — docker exec MySQL 한글 이중 인코딩
- [[docker-compose-stale-path-bug]] — 저장소 분리 후 docker-compose 볼륨 경로가 안 맞던 문제
- [[terraform-circular-module-dependency]] — 참고 레포(PAPERPLE-INFRA)의 vpc↔subnet 순환 참조
- [[github-actions-oidc-not-authorized]] — GitHub Actions OIDC "Not authorized" (근본 원인 미해결, sub+와일드카드로 우회)
- [[eks-provider-auth]] — kubernetes/helm/kubectl provider의 EKS 인증 실패 (토큰 만료 + role-arn 누락)
- [[eks-destroy-layer-separation]] — EKS destroy 시 K8s addon 정리 실패 종합 정리 + Layer 1(infrastructure)/Layer 2(k8s-addon) 분리 해법 (구현 완료)
- [[cd-helm-chart-deploy-review]] — CD Helm 차트 실동작 리뷰에서 찾은 문제 4가지(frontend 백엔드 주소 누락, ArgoCD path 오류, frontend Dockerfile 중복 빌드, backend 필수 env var 누락) — OAuth/Toss 시크릿은 이후 해결됨([[decisions/2026-08-11-external-api-secrets-manager]]), 문제 1의 `rewrites()` 방식은 이후 폐기됨([[decisions/2026-08-11-frontend-api-routing-alb-not-rewrites]])
- [[destroy-order-incident-and-webhook-orphans]] — 실제로 destroy 순서를 어겨서 겪은 사고: IAM 손상 복구, ALB/TG/SG 고아 리소스 정리, ALB Controller·ESO 둘 다에서 재현된 "죽은 컨트롤러의 admission webhook이 finalizer를 영구히 막는" 문제
- [[alb-ingressgroup-orphan-on-rename]] — ALB Controller가 Ingress의 `group.name`을 바꿔도 옛 IngressGroup(ALB/TG)을 자동으로 안 치움
- [[backend-ci-missing-service-containers]] — backend CI에 MySQL/Redis 서비스 컨테이너가 없어서 `contextLoads` 테스트 실패, GitHub Actions `services:`로 해결
- [[payment-eso-secret-staleness]] — Toss 결제 400 에러, 원인은 페어링 불일치가 아니라 ESO 1시간 재동기화 주기 동안 K8s Secret에 남아있던 stale한 값
- [[ses-dkim-preexisting-records-import]] — `03_registry` 첫 apply에서 SES DKIM CNAME이 이미 존재해 실패, `terraform import`로 기존 레코드를 state에 편입
- [[lambda-env-var-and-runtime-version-gotchas]] — Lambda 예약 환경변수(`AWS_REGION`) 직접 설정 불가, `nodejs24.x`는 AWS provider v6.21.0+ 필요(현재 `~> 5.0` 고정이라 업그레이드 전까지 `nodejs22.x` 사용)
- [[grafana-amp-datasource-missing-auth-token]] — Grafana의 `grafana-amazonprometheus-datasource` 플러그인이 쿼리 요청에 인증 헤더를 안 붙이는 문제 — **해결됨(2026-08-18)**, `jsonData.sigV4Auth` 필드 누락이 원인이었음
- [[grafana-amp-rate-interval-sparse-historical-data]] — AMP 전환 후 넓은 시간 범위로 보면 예전 데이터가 빈 값으로 보이는 문제 — **해결됨(2026-08-18)**, `rate()`/`increase()`의 고정 `[5m]` 윈도우를 `$__rate_interval`로 교체
- [[keda-scaling-missing-metrics-server]] — 부하테스트에서 KEDA(CPU 트리거)가 전혀 스케일 안 함, 원인은 클러스터에 `metrics-server` 자체가 없었던 것(HPA Condition까지 봐야 드러남) — 설치로 해결, replica 4→8 실제 증가 검증
- [[hikaricp-connection-storm-load-test]] — 부하테스트만 돌리면 ArgoCD/Grafana까지 같이 멈춤, 원인은 RDS가 아니라 HikariCP 풀 과다(40×8replica=320)로 인한 커넥션 생성 시 CPU 폭증이 버스터블(t3.medium) 노드의 CPU 크레딧을 고갈시켜 같은 노드의 다른 파드까지 굶긴 것 — 풀 사이즈 축소로 해결
- [[backend-cpu-throttling-and-scaling-load-test]] — backend CPU 스로틀링 2단계: 1차는 CPU limit 자체가 작아서(250m/1) 상향으로 해결(500m/2), 2차는 limit 상향 후에도 순간 스로틀링이 남아있는데 KEDA는 정상적으로 안 늘리는 상황을 관찰 — "스로틀링≠평균 사용률" 구조를 확인하고 replica 증설 대신 limit 추가 상향을 권고(**미적용**, 사용자 판단 대기)
- [[frontend-cpu-throttling-cfs-quota-vs-jvm-tradeoff]] — frontend KEDA 적용 후 유휴 부하에서도 스로틀링 지속 발견, CFS 100ms 쿼터 + Node.js 보조 스레드(GC/libuv/JIT) 버스트가 원인임을 실측(Prometheus 직접 질의)으로 확인 — frontend는 CPU limit을 아예 제거(request만 유지)해서 해결, backend는 JVM 컨테이너 인지 방식(`ActiveProcessorCount` 자동 사이징) 때문에 같은 처방을 안 씀. 수정이 한동안 CD 레포에 커밋만 되고 미배포 상태였던 것도 뒤늦게 발견·정리(**해결됨**)
- [[2026-08-18-config-build-bugs]] — 같은 날 겪은 독립적 설정/빌드 버그 3건: `03_registry` output이 존재하지 않는 `module.ses` 참조하던 문제, `notification-config`라는 실체 없는 ConfigMap을 Helm values가 참조해서 backend가 `CreateContainerConfigError`로 못 뜨던 문제, backend Dockerfile이 Ubuntu Noble 베이스 전환으로 `useradd --uid 1000`이 충돌하던 문제(`-jammy` 태그 고정으로 해결)
- [[loadtest-10000-open-run-cascading-failures]] — 만 명 규모 오픈런 부하테스트 실행기: k6 클라이언트 함정 3개(`constant-vus`가 VU 수를 부풀리는 문제, macOS TLS 검증이 k6 프로세스를 크래시시키는 문제, VU 1만 개 동시 접속이 ALB TLS negotiation 자체를 실패시키는 문제) 전부 해결하고 나서 발견한 **진짜 서버 문제** — 부하가 심해지면 파드가 헬스체크에도 응답 못 해서 ALB가 로테이션에서 제외, 도미노로 전부 제외되며 HealthyHostCount=0(완전 다운)까지 재현됨. ArgoCD repo-server도 BestEffort QoS라 같은 부하에 굶어 죽는 것도 같이 발견. **2026-08-19 후속**: 재현 중 애플리케이션 레벨 원인 2건(backend actuator가 일반 요청과 스레드풀 공유, frontend SSR 캐싱 전무) 추가 특정해서 GitHub 이슈 3건 등록([qKet/frontend#27](https://github.com/qKet/frontend/issues/27), [#28](https://github.com/qKet/frontend/issues/28), [qKet/backend#29](https://github.com/qKet/backend/issues/29)) — **셋 다 미착수, ArgoCD repo-server 건은 이슈도 아직 없음**

## runbook/ — 반복 운영 절차
- [[db-schema-change]] — 로컬 DB 스키마 변경 절차 2가지
- [[terraform-apply-order]] — Terraform 최초 적용 절차 (infrastructure → k8s-addon → registry/data, 구 platform → workload)
- [[daily-infrastructure-toggle]] — 매일 아침/저녁 `01_infrastructure`/`02_k8s-addon` 켜고 끄기 (`Infra/README.md`에 명령어 정리, `04_data`는 AWS 리소스는 안 건드리지만 K8s 오브젝트 재생성 위해 아침에 apply 한 번 더 필요)

---

## 고아 페이지 / 미해결 항목

- CI/CD 나머지 설계(재사용 워크플로우, `qKet/CD` 레포 구조, ArgoCD Application, Terraform `modules/argocd`)는 아직 실제 파일로 안 만들어짐 — [[2026-08-06-ci-tool-github-actions]] 참고
- [[github-actions-oidc-not-authorized]]의 근본 원인 미해결 — 커스텀 claim(`repository`/`ref`/`job_workflow_ref`)이 왜 안 먹혔는지 원인 규명 못 함 (ID 와일드카드 트레이드오프는 정확한 ID로 교체해서 해소됨)
- `modules/irsa`, `modules/eks/access.tf`(cluster_admin role), `modules/github-actions-oidc`, ArgoCD helm_release 등 최근 추가된 Terraform 리소스들이 아직 architecture/decisions 문서로 안 남겨짐
- `Infra/kubernetes/{release,prod}/namespace_qKet.yaml`이 `kubernetes_namespace.qket`(infrastructure)과 중복 — 삭제는 보류하기로 함(2026-08-10), ArgoCD "infra-manifests" Application을 실제로 만들 때 다시 정리하기로 함
- `02_k8s-addon` root 신설 완료(2026-08-10) — namespace/ArgoCD를 `01_infrastructure`에서 분리함. [[troubleshooting/eks-destroy-layer-separation]] 참고
- `03_registry` root 신설 완료(2026-08-10) — ECR/github-actions-oidc를 `01_infrastructure`에서 분리함
- ✅ (갱신) ESO(External Secrets Operator)는 2026-08-10에 이미 재활성화됨(`04_data`에서 `module.eso`로 db-secrets/redis-secrets/external-api-secrets 전부 동기화 중) — 이 줄이 옛날에 "여전히 backup/에 보류 중"이라고 남아있던 건 드리프트, 이번에 바로잡음
- ✅ (해결) [[cd-helm-chart-deploy-review]]의 OAuth 3사/`TOSS_SECRET_KEY` 미연결 문제 — [[decisions/2026-08-11-external-api-secrets-manager]]로 해결
- prod용 ArgoCD Application이 아직 없음 — release용(`Infra/argocd/qket-cd-app.yaml`)만 있고, prod는 `valueFiles: [values.yaml, values-prod.yaml]` 레이어링이 필요함 ([[cd-helm-chart-deploy-review]] 참고)
- ✅ (대부분 해결, 2026-08-12) [[decisions/2026-08-11-monitoring-stack-design]] — `module.monitoring`(kube-prometheus-stack) 실제 apply, 클러스터 지표+CloudWatch DB 연동+Grafana 노출, backend ServiceMonitor(release만), 대시보드 git+ConfigMap 영구저장까지 완료. 지표 영구저장은 문서상 EBS 방침에서 **AMP**(Amazon Managed Prometheus)로 방향을 바꿔서 remoteWrite 검증까지 완료(EBS는 `ReclaimPolicy: Delete`라 EKS destroy 시 볼륨까지 같이 사라져서 "클러스터 재생성에도 유지"라는 목표 자체를 못 푸는 걸 확인하고 기각). Grafana→AMP 직접 조회는 2026-08-18 해결됨([[troubleshooting/grafana-amp-datasource-missing-auth-token]]). 여전히 미해결: Loki(로그), 알림 채널(Slack/이메일), Blackbox 외부 헬스체크, ALB 에러율 연동, 배포 annotation
- 2026-08-11에 `02_k8s-addon` 실제 apply/destroy 중 겪은 문제들(ALB Controller destroy 순서, kubectl_manifest가 finalizer를 안 기다리는 문제, ArgoCD/ExternalDNS와 ALB Controller의 webhook 레이스, ExternalDNS TXT 소유권 없는 수동 레코드 방치)이 아직 troubleshooting 문서로 안 남겨짐 — 필요하면 추가 정리 예정
- [[decisions/2026-08-11-vpn-access-control-paused]] — dev/Grafana/CD를 VPN 필수로 바꿀지 논의만 하고 팀 상의를 위해 보류됨. **팀 결정 없이 임의로 재개하지 말 것**
- `modules/messaging`의 Lambda 코드(`lambda-src/index.mjs`)는 팀원이 작성한 실제 코드로 교체 완료했지만, 실제 배포 후 end-to-end 발송 테스트(SQS에 메시지 넣어서 진짜 메일이 오는지)는 아직 안 해봄
- 콘솔에서 예전에 수동으로 만들어져 있던 `qket-email-verification-lambda`(Terraform 미관리) — 새 Terraform 관리 함수(`team5-qket-email-verification-release`)가 정상 동작 확인되면 정리 필요
- `nodejs24.x`를 쓰려면 AWS provider `~> 5.0` → v6.21.0+ 메이저 업그레이드가 필요함(4개 root 전부 영향) — 아직 안 함, 당장은 `nodejs22.x`로 유지 중. [[lambda-env-var-and-runtime-version-gotchas]] 참고
- 이메일 발송 인프라(`external_api`, `messaging`)는 release workspace에만 apply됨 — prod workspace는 아직 한 번도 적용 안 함
- SES 발신 도메인(`jun979.click`)에 SPF TXT 레코드가 아직 없음 — `v=spf1 include:amazonses.com ~all` 추가 필요, 담당자 미지정 ([[architecture/messaging-infrastructure]] 참고)
- Lambda 코드가 아직 이메일 인증번호(`{ email, code }`)만 처리 — 예매확정/취소 알림(`type` 필드 분기)은 설계만 있고 실제 코드 반영은 안 됨
- EventBridge Scheduler를 이미 만들어둔 팀원이 있다는 메모가 있음 — 실제 용도(리마인더 등) 확인 필요, 확정된 요구사항이 없다면 정리 대상인지 판단 필요 ([[decisions/2026-08-12-sqs-lambda-ses-notification-pipeline]] 참고)
- 프론트 CI의 Toss 키 fetch가 `team5-qket-external-api-release` 이름으로 하드코딩됨 — prod CI 파이프라인을 만들 때 반드시 갱신 필요 ([[decisions/2026-08-11-frontend-ci-toss-key-secrets-manager]] 참고)
- [[decisions/2026-08-11-frontend-api-routing-alb-not-rewrites]]로 배포 환경 라우팅은 바뀌었지만, 로컬 개발(`npm run dev`) 환경에서 `/api` 프록시를 어떻게 처리할지는 아직 정리 안 됨(온보딩 문서에도 미반영)
- ✅ (해결, 2026-08-18) [[troubleshooting/backend-cpu-throttling-and-scaling-load-test]]의 2차 권고(backend CPU limit 추가 상향) — `CD/helm/values.yaml`의 `backend.resources.limits.cpu`가 `2`→`3`으로 반영됨, `backend.autoscaling.maxReplicas`도 `8`→`12`로 상향됨(만 명 부하테스트 중 CPU가 목표치를 넘어도 8개가 상한이라 못 늘어나는 걸 확인하고 조치)
- 🔴 (신규, 2026-08-18, 최우선) ALB 타겟그룹 헬스체크 연쇄 장애 — 만 명 부하테스트에서 backend/frontend 둘 다 `HealthyHostCount=0`(완전 다운)까지 재현됨. frontend 헬스체크 경로가 무거운 SSR 페이지(`/`)를 그대로 쓰는 게 직접 원인. [[troubleshooting/loadtest-10000-open-run-cascading-failures]] "문제 4" 참고 — **미해결, 다음 세션에서 이어서**
- 🔴 (신규, 2026-08-18) ArgoCD `repo-server`가 CPU/메모리 request 없음(BestEffort) — 부하테스트 중 노드가 바빠지자 자기 헬스체크에도 응답 못 해 재시작당함, ArgoCD UI가 일시적으로 "connection refused" 뱉음. [[troubleshooting/loadtest-10000-open-run-cascading-failures]] 참고 — **미해결**
- ✅ (대부분 해결, 2026-08-18) [[decisions/2026-08-18-capacity-planning-large-traffic-readiness]] — cluster-autoscaler 설치, RDS release를 `db.t3.medium`으로 상향, 500~10000명 규모 재측정까지 진행 완료. 여전히 미착수/미해결: prod RDS 상향(release 결과 보고 결정 예정), Redis 용량/HA(사용자가 명시적으로 보류), RDS 버스터블 크레딧 장시간 소진 시나리오, 위 ALB 헬스체크 연쇄 장애
- `Infra` PR #15(`feature/nyj`, 모니터링/AMP 작업)가 merge conflict 해결 완료로 `MERGEABLE` 상태까지 갔지만 아직 `dev`로 실제 merge는 안 됨 — 나윤준님 직접 확인 후 merge할지, 바로 merge할지 미결정
- 🔴 (신규, 2026-08-19, 미문서화) `02_k8s-addon/argocd-notifications-eso.tf`(ArgoCD 알림 이메일 기능, 커밋 `4e7a965`)가 `04_data`의 ESO가 설치하는 `external-secrets.io/v1` CRD를 참조하는데, 확립된 apply 순서(`infrastructure→k8s-addon→data`)와 반대 방향 의존이라 매일 아침 처음 apply할 때마다 CRD 못 찾는 에러 재현 예상(어제 ServiceMonitor CRD와 같은 계열, 이번엔 root 경계를 넘음). 우회법(`04_data`의 `module.eso`를 먼저 targeted apply)은 찾았지만 아직 정식 troubleshooting 문서로 안 남겨짐 — 다음에 또 겪으면 그때 문서화할 것
