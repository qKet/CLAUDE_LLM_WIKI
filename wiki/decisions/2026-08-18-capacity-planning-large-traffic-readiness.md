---
title: 대용량 트래픽 대비 현재 인프라 용량 분석 — 노드 오토스케일러 부재가 가장 큰 병목
category: decisions
status: 논의중 (분석/권고만 완료, 실제 조치는 아직 안 함)
date: 2026-08-18
author: Claude Code
tags: [capacity-planning, autoscaling, cluster-autoscaler, karpenter, rds, redis, keda]
updated: 2026-08-18
---

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

## 관련
- [[../architecture/keda-autoscaling]]
- [[../troubleshooting/frontend-cpu-throttling-cfs-quota-vs-jvm-tradeoff]]
- [[../troubleshooting/hikaricp-connection-storm-load-test]]
- [[../troubleshooting/backend-cpu-throttling-and-scaling-load-test]]
- [[2026-08-10-redis-session-queue-shared-instance-risk]]
