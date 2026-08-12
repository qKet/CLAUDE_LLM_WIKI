# Index

Qket 프로젝트 팀 위키의 전체 페이지 목록. 새 페이지를 추가하면 여기 등록한다.

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
- [[terraform-platform-workload-split]] — infrastructure(공유)/data(workspace) 2-root 구조(구 platform/workload), env_config 패턴

## decisions/ — "왜 이렇게 하기로 했나" (ADR)
- [[README]] — 사용법/템플릿
- [[2026-08-06-ci-tool-github-actions]] — CI 도구로 GitHub Actions 선택 (Jenkins 대신)
- [[2026-08-06-terraform-module-restructure]] — Terraform 모듈 리소스타입별 재편 + platform/workload 2-root화
- [[2026-08-06-ecr-recreate-vs-import]] — 기존 ECR 저장소 import 대신 삭제 후 재생성
- [[2026-08-10-redis-session-queue-shared-instance-risk]] — 세션/대기열이 같은 Redis 인스턴스 공유 시 리스크(noisy neighbor, 메모리 압박), 페일오버로도 안 풀리는 부분, 물리 분리 vs 논리 DB 분리 비용 비교 (논의중 — 부하테스트 후 재검토)
- [[2026-08-10-cd-writeback-github-app]] — CD 레포 write-back 인증으로 GitHub App 선택(PAT 대신) — 장기 자격증명을 시크릿에 안 두려는 원칙, 개인 계정으로 잘못 생성됐던 시행착오 포함
- [[2026-08-11-monitoring-stack-design]] — 모니터링 스택 설계(kube-prometheus-stack + Loki + CloudWatch 연동), 로그/지표 영구 저장소를 `03_registry`에 두기로 한 이유, 1차/2차 범위 구분 (논의중 — 코드 구현 전, 알림 채널 미정)

## troubleshooting/ — 실제 겪은 버그
- [[null-field-partial-update-bug]] — nullable FK 부분 업데이트 버그 (백/프론트 양쪽)
- [[utf8mb4-encoding-bug]] — docker exec MySQL 한글 이중 인코딩
- [[docker-compose-stale-path-bug]] — 저장소 분리 후 docker-compose 볼륨 경로가 안 맞던 문제
- [[terraform-circular-module-dependency]] — 참고 레포(PAPERPLE-INFRA)의 vpc↔subnet 순환 참조
- [[github-actions-oidc-not-authorized]] — GitHub Actions OIDC "Not authorized" (근본 원인 미해결, sub+와일드카드로 우회)
- [[eks-provider-auth]] — kubernetes/helm/kubectl provider의 EKS 인증 실패 (토큰 만료 + role-arn 누락)
- [[eks-destroy-layer-separation]] — EKS destroy 시 K8s addon 정리 실패 종합 정리 + Layer 1(infrastructure)/Layer 2(k8s-addon) 분리 해법 (구현 완료)
- [[cd-helm-chart-deploy-review]] — CD Helm 차트 실동작 리뷰에서 찾은 문제 4가지(frontend 백엔드 주소 누락, ArgoCD path 오류, frontend Dockerfile 중복 빌드, backend 필수 env var 누락) — OAuth/Toss 시크릿은 실제 발급값 필요해서 미해결
- [[grafana-amp-datasource-missing-auth-token]] — Grafana의 `grafana-amazonprometheus-datasource` 플러그인이 쿼리 요청에 인증 헤더를 안 붙이는 문제(IRSA/access key 둘 다 동일 실패, AMP API/IAM은 정상 확인됨) — **미해결**, remoteWrite(데이터 저장)는 정상 작동 중

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
- ESO(External Secrets Operator)만 여전히 `Infra/backup/`에 보류 중 — ALB Controller는 2026-08-10에 `02_k8s-addon`으로 재활성화 완료
- [[cd-helm-chart-deploy-review]]의 OAuth 3사(Google/Kakao/Naver) client-id/secret + `TOSS_SECRET_KEY`가 CD 차트에 안 연결돼 있음 — 실제 발급값이 준비되면 ESO 패턴으로 새 Secret 추가 필요
- prod용 ArgoCD Application이 아직 없음 — release용(`Infra/argocd/qket-cd-app.yaml`)만 있고, prod는 `valueFiles: [values.yaml, values-prod.yaml]` 레이어링이 필요함 ([[cd-helm-chart-deploy-review]] 참고)
- [[decisions/2026-08-11-monitoring-stack-design]] 중 상당 부분 구현됨(2026-08-12): `module.monitoring`(kube-prometheus-stack) 실제 apply, backend ServiceMonitor(release만), 대시보드 git+ConfigMap 영구저장, 지표는 EBS 대신 **AMP**(Amazon Managed Prometheus)로 방향 전환해서 영구저장까지 완료(remoteWrite 검증됨). 남은 미해결: Grafana→AMP 직접 조회([[troubleshooting/grafana-amp-datasource-missing-auth-token]]), 알림 채널(Slack/이메일) 미정, Loki(로그) 미구현
- 2026-08-11에 `02_k8s-addon` 실제 apply/destroy 중 겪은 문제들(ALB Controller destroy 순서, kubectl_manifest가 finalizer를 안 기다리는 문제, ArgoCD/ExternalDNS와 ALB Controller의 webhook 레이스, ExternalDNS TXT 소유권 없는 수동 레코드 방치)이 아직 troubleshooting 문서로 안 남겨짐 — 필요하면 추가 정리 예정
