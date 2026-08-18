---
title: 대용량 트래픽 대비 현재 인프라 용량 분석 — 노드 오토스케일러 부재가 가장 큰 병목
category: decisions
status: 부분 구현 완료 (권고 1·2 반영, 3은 사용자 판단으로 보류, 4는 실행 중 — 새 심각 리스크 발견)
date: 2026-08-18
author: Claude Code
tags: [capacity-planning, autoscaling, cluster-autoscaler, karpenter, rds, redis, keda, alb, health-check]
updated: 2026-08-18
---

> ✅ 2026-08-18 같은 날 바로 후속 조치까지 진행: 권고 1(cluster-autoscaler)과 2(RDS 상향, release만)를 실제로 적용함. 권고 3(Redis)은 "대기열 기능 구현 시 물리 분리가 필요할 수 있다"는 이유로 사용자가 명시적으로 보류 결정([[2026-08-10-redis-session-queue-shared-instance-risk]] 참고). 권고 4(장시간 재측정)는 500명→만 명까지 실제로 진행했고, 그 과정에서 이 문서가 예측 못 했던 **새로운 심각한 리스크(ALB 헬스체크 연쇄 장애로 전체 다운)** 를 발견함 — [[../troubleshooting/loadtest-10000-open-run-cascading-failures]] 참고, 아직 미해결. 아래 원본 분석/권고는 그대로 두고, 실제로 뭘 했는지는 맨 아래 "구현 결과" 섹션에 정리.

# 대용량 트래픽 대비 현재 인프라 용량 분석

## 배경

"대용량 트래픽이 몰리면 지금 CPU/메모리 스펙으로 안전한가?"라는 질문에 답하기 위해, 추측이 아니라 클러스터에 직접 접속해서 실측 데이터로 분석함. `frontend`/`backend` CPU limit 튜닝([[../troubleshooting/frontend-cpu-throttling-cfs-quota-vs-jvm-tradeoff]]) 작업 직후 이어서 진행.

## 조사 방법 (실측)

- `kubectl get nodes` / `kubectl describe node` — 노드 스펙, allocatable, 현재 할당률
- `kubectl top nodes` — 실시간 사용률
- Terraform 코드(`01_infrastructure/variables.tf`, `modules/rds`, `modules/redis`, `04_data/main.tf`) — 인스턴스 클래스/오토스케일링 범위 확인
- 리포 전체에서 `cluster-autoscaler`/`karpenter` 검색 — 매칭 없음

## 발견 사항

### 1. 🔴 가장 심각: 노드 오토스케일러가 아예 없음

`01_infrastructure/variables.tf`에 `node_desired_size=2`/`node_min_size=1`/`node_max_size=3`(t3.xlarge)가 정의돼 있지만, 이 `max_size`는 **cluster-autoscaler나 Karpenter 같은 컨트롤러가 있어야만 실제로 쓰이는 상한선**임. 리포 전체(`Infra/`)를 검색해도 이 둘 중 아무것도 설치돼 있지 않음.

즉 EKS 관리형 노드그룹은 desired_size(2)에 고정된 채 **스스로 늘어나지 않음**. KEDA([[../architecture/keda-autoscaling]])가 `backend`/`frontend` 파드를 각각 4→8까지 늘리라고 명령할 수는 있지만, 그 파드들을 실제로 얹을 노드가 없으면 그냥 `Pending`으로 멈춤 — **파드 레벨 오토스케일링(KEDA)은 이미 준비돼 있는데, 그 밑을 받치는 노드 레벨 오토스케일링이 없어서 무의미해지는 상황**.

### 2. 노드 CPU: 이미 baseline에서 여유가 많지 않음

실측(`kubectl describe node`, backend/frontend가 최소 replica 4/4일 때):

```
node1: cpu 2550m / 3920m allocatable (65%)
node2: cpu 2350m / 3920m allocatable (60%)
```

이건 ArgoCD, ALB Controller, kube-prometheus-stack(Prometheus/Grafana/Alertmanager/node-exporter), KEDA, ESO, metrics-server 등 애드온까지 다 포함된 baseline. `backend`/`frontend`가 각각 max(8/8)까지 스케일아웃하면 request 기준으로 추가 약 +4000m(각 +2000m)가 더 필요한데, 지금 2노드 allocatable 총합(7840m)로는 부족함 — 계산상 8900m 필요, 2노드로는 충당 불가. 위 1번 문제(오토스케일러 부재) 때문에 3번째 노드도 자동으로 안 생겨서 그대로 병목.

메모리는 반대로 여유가 큼(노드당 10~12%만 사용 중) — 이번 분석에서 CPU/노드 오토스케일링만 실질적 병목이고 메모리는 당장 문제 아님.

### 3. RDS 커넥션 여유가 얇음 (기존에 이미 알려진 리스크, 재확인)

`db.t3.micro`(release) 기준 `max_connections ≈ 85`. `backend.dbPoolSize(10) × maxReplicas(8) = 80` — 여유가 5뿐. `2026-08-13 부하테스트 사고`([[../troubleshooting/hikaricp-connection-storm-load-test]])에서 이미 이 계산의 중요성이 드러난 바 있음, 지금도 그대로.

추가로: `db.t3.micro`는 **버스터블 인스턴스**라 CPU 크레딧이 있음. k6 스크립트(`spike_1000_login.js`, 90초 스파이크)처럼 짧은 테스트는 크레딧으로 버티지만, 실제 오픈런처럼 **몇 분 이상 지속되는 대용량 트래픽**에서는 크레딧 소진 후 급격한 성능 저하 위험이 있음 — 이 부분은 아직 실측 안 됨(90초짜리 스크립트로는 안 드러남).

### 4. Redis: 단일 노드, 소용량 (기존 리스크와 연결)

`cache.t3.micro`, `num_cache_nodes=1`(복제/장애조치 없음). [[2026-08-10-redis-session-queue-shared-instance-risk]]에서 이미 세션/대기열 공유로 인한 리스크가 논의됐었는데, 대용량 트래픽 관점에서는 여기에 **용량 자체가 작다**(0.5GiB급)는 문제가 추가됨.

## 결정

**아직 없음 — 이 문서는 분석/권고까지만.** 실제 조치는 사용자 판단 대기 중.

## 권고 우선순위 (미적용)

1. **cluster-autoscaler(또는 Karpenter) 설치** — 다른 모든 대응보다 선행돼야 함. 이게 없으면 KEDA의 `maxReplicas`도, RDS/Redis 스펙 상향도 "노드가 없어서 파드가 안 뜨는" 근본 문제를 안 풀어줌
2. RDS 인스턴스 클래스 상향(`db.t3.micro` → `db.t3.small` 이상, 또는 버스터블 아닌 클래스) + `dbPoolSize`/`maxReplicas` 재계산
3. Redis 용량/HA 재검토
4. 목표 동접자 수 기준으로 k6 시나리오 규모/지속시간을 키워서 재측정 — 현재 `spike_1000_login.js`는 1000 VU/90초 스파이크만 검증했고, "대용량이 몇 분 이상 지속"되는 시나리오는 아직 실측 데이터가 없음

## 고려한 대안

- **노드 인스턴스 타입을 더 큰 것으로 바꾸는 것(수직 확장)** vs **오토스케일러로 노드 수를 늘리는 것(수평 확장)** — 이 문서에서는 수평 확장(오토스케일러)을 1순위로 권고함. 이유: 현재 baseline도 이미 60%대라 인스턴스 하나를 키우는 것보단 트래픽에 비례해서 노드 자체가 늘고 주는 구조가 비용(야간/평시엔 노드 최소, 오픈런 때만 최대)과 안전성 둘 다에 유리

## 트레이드오프 / 남은 리스크

- cluster-autoscaler/Karpenter 도입은 그 자체로 새로운 운영 부담(스케일 다운 타이밍, 파드 disruption budget 등)이 생김 — 도입 시 별도 architecture 문서 필요
- RDS 버스터블 CPU 크레딧 소진 시나리오는 아직 실측 없이 이론적 위험으로만 기록함 — 실제 장시간 부하테스트로 검증 필요

## 구현 결과 (실제로 한 것, 2026-08-18)

### 1. cluster-autoscaler 설치 완료

`Infra/modules/addons/cluster-autoscaler` 신설(IRSA + `helm_release`, `02_k8s-addon`에서 호출) — 상세 구조는 [[../architecture/cluster-autoscaler]] 참고. 설치 직후 파드가 IRSA trust policy 전파 지연으로 1번 크래시했다가 자동 재시작 후 정상화되는 걸 실측함(재발 예상, 조치 불필요 — 위 문서 참고).

### 2. RDS 인스턴스 클래스 상향 완료 (release만)

`04_data/main.tf`의 `env_config_map.release.db_instance_class`를 `db.t3.micro` → `db.t3.medium`으로 변경, `terraform apply` 완료. **주의**: `aws_db_instance`는 `apply_immediately`를 명시적으로 켜두지 않아서(모듈에 없음), `terraform apply`가 성공해도 실제 변경은 기본적으로 **다음 유지보수 기간까지 미뤄짐**(`PendingModifiedValues`로만 반영됨, 실제 클래스는 그대로) — 즉시 반영하려면 `aws rds modify-db-instance --apply-immediately`를 따로 호출해야 함(짧은 재부팅 발생, 1~2분). 이번엔 사용자 확인 후 즉시 적용까지 완료해서 실제로 `db.t3.medium`으로 전환됨.

prod는 아직 `db.t3.small` 그대로 — release에서 재측정 후 결정하기로 함(사용자 판단).

### 3. Redis — 보류 (사용자 결정)

용량/HA 상향 안 함. 이유는 [[2026-08-10-redis-session-queue-shared-instance-risk]]에 추가된 2026-08-18 노트 참고 — 지금 스펙만 올려봐야 재검토 트리거(부하테스트 실측)가 없어서 근거 없는 비용 증가이고, 나중에 대기열 기능을 실제로 만들 때 세션/대기열 물리 분리(그 문서의 대안 A)로 갈 가능성이 있어 그때 한 번에 정하는 게 낫다고 판단.

### 4. 목표 규모 재측정 — 500명 → 만 명까지 진행, 새로운 리스크 발견

`Infra/loadtest/open_run_10000_no_queue.js`/`open_run_10000_with_queue.js`(신설)로 500 → 10000명까지 단계적으로 올려서 재측정함. 자세한 진단 과정과 발견 전부는 [[../troubleshooting/loadtest-10000-open-run-cascading-failures]] 참고, 여기는 이 문서(용량 분석)와 직결되는 결론만 요약:

- **cluster-autoscaler/KEDA 실전 검증 완료** — 500명 테스트 중 실제로 3번째 노드가 자동으로 붙는 것, KEDA가 backend/frontend를 4→8로 스케일아웃하는 것 전부 실측으로 확인됨(이 문서 1번 발견이 실제로 해결됐음이 증명됨)
- **KEDA는 반응형이라 순간 폭증엔 근본적 한계가 있음(신규 발견)** — 트래픽이 "그 순간 다 몰리면" 판단→새 파드 스케줄링→JVM 부팅까지의 지연(수십 초~1분)을 오토스케일러가 못 없애줌. 그 사이엔 기존 최소 replica(4개)가 부하를 몰빵으로 받음("thundering herd") — 대기열이나 사전 스케일업 같은 별도 장치가 필요, 오토스케일러 하나로는 못 풂
- **backend KEDA `maxReplicas`를 8→12로 상향함** — 실측 중 CPU가 목표(70%)를 넘어 102%까지 갔는데도 8개가 상한이라 더 못 늘어나는 걸 확인. RDS가 `db.t3.medium`(~340 커넥션)으로 이미 올라가 있어서 `12 × dbPoolSize(10) = 120`도 안전한 여유 범위. `backend.resources.limits.cpu`도 이 과정에서 `2` → `3`으로 같이 상향됨
- **🔴 새로 발견된 심각한 리스크: ALB 헬스체크 연쇄 장애로 완전 다운** — 이 문서가 예상 못 했던 층위의 문제. 부하가 심해지면 파드가 헬스체크에도 제때 응답 못 해서 ALB가 로테이션에서 제외하고, 남은 파드가 더 몰려서 도미노로 전부 제외되며 `HealthyHostCount=0`(완전 다운)까지 실제로 재현됨. **이 문서의 "노드/DB/Redis 스펙" 관점 분석만으로는 못 잡아내는 종류의 리스크였음** — 자세한 내용/미해결 대응 방안은 [[../troubleshooting/loadtest-10000-open-run-cascading-failures]]의 "문제 4" 참고
- RDS 버스터블 CPU 크레딧 소진 시나리오는 여전히 미검증(이번 테스트도 순간 폭증형이라 "몇 분 지속" 시나리오는 아직 없음)

## 다음 세션에서 이어갈 것 (2026-08-18 종료 시점 기준)

1. **최우선**: frontend 헬스체크 경로를 무거운 `/`에서 전용 엔드포인트로 분리 — ALB 연쇄 장애의 직접 원인
2. ArgoCD `repo-server`에 최소 리소스 request 부여(BestEffort라 부하테스트 중 자기가 먼저 굶어 죽는 걸 확인함)
3. 위 조치 후 `open_run_10000_with_queue.js`(대기열 포함판)로도 같은 규모 재측정 — 대기열이 이 연쇄 장애를 실제로 막아주는지 검증
4. RDS 버스터블 크레딧 소진 여부는 여전히 미검증 — 짧은 스파이크가 아니라 몇 분 이상 지속되는 시나리오 필요

## 관련
- [[../architecture/keda-autoscaling]]
- [[../architecture/cluster-autoscaler]]
- [[../troubleshooting/frontend-cpu-throttling-cfs-quota-vs-jvm-tradeoff]]
- [[../troubleshooting/hikaricp-connection-storm-load-test]]
- [[../troubleshooting/backend-cpu-throttling-and-scaling-load-test]]
- [[../troubleshooting/loadtest-10000-open-run-cascading-failures]]
- [[2026-08-10-redis-session-queue-shared-instance-risk]]
