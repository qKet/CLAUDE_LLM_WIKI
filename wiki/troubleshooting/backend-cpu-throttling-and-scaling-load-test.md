---
title: 부하테스트 중 backend CPU 스로틀링 — 1차(limit 상향)와 2차(replica vs limit 재판단)
category: troubleshooting
tags: [monitoring, cpu-throttling, keda, autoscaling, cpu-limit, grafana, prometheus, load-test]
created: 2026-08-13
updated: 2026-08-18
---

# 부하테스트 중 backend CPU 스로틀링 — 1차(limit 상향)와 2차(replica vs limit 재판단)

> 2026-08-18: frontend에도 같은 계열의 스로틀링이 발견됐는데, 원인 구조는 같아도(CFS 100ms 쿼터) 처방은 정반대(backend는 limit 유지, frontend는 limit 제거)였음 — Node.js와 JVM의 컨테이너 CPU 인지 방식 차이 때문. [[frontend-cpu-throttling-cfs-quota-vs-jvm-tradeoff]] 참고. 같은 날 실측(유휴~저부하 트래픽 기준)으로는 backend 스로틀링이 정확히 0으로 확인되어, 아래 2차 권고(limit 2→3~4 추가 상향)의 긴급성은 낮아졌으나 실제 부하테스트로 재검증된 건 아님.

## 1차: CPU limit이 너무 작았던 문제

### 증상

k6로 1000명 동시 로그인 부하테스트 시 응답이 눈에 띄게 느려짐.

### 원인 진단

Prometheus에서 `container_cpu_cfs_throttled_periods_total`(cAdvisor 지표, CPU 스로틀링의 결정적 신호 — 단순 사용률과 다름)을 조회:

```
sum(increase(container_cpu_cfs_throttled_periods_total{namespace="qket-release", container="qket-backend"}[5m])) by (pod)
```

5분간 1500~2300 periods 수준으로 심하게 스로틀링되고 있었음. 당시 backend의 CPU 리소스가 `request: 250m` / `limit: 1` 이었는데, 로그인 시 BCrypt 해싱 연산이 CPU를 상당히 쓰는데 1000명 동시 요청 앞에서 이 limit이 너무 작았음.

### 해결

`CD/helm/values.yaml`의 `backend.resources`를 상향:

```yaml
resources:
  requests: { cpu: "500m", memory: "512Mi" }
  limits:   { cpu: "2",    memory: "1Gi" }
```

### 검증

재테스트에서 스로틀링 수치가 크게 줄고 응답 시간도 개선됨.

## 2차: 신규 Grafana 패널로 재관찰 — "그럼 replica를 더 늘려야 하나?"

### 배경

자가진단이 가능하도록 Grafana 대시보드(`02_k8s-addon/dashboards/qket-monitoring.json`)에 패널 2개를 추가함:
- **"CPU 스로틀링 (5분간, pod별)"** — 위 PromQL 그대로, pod별로 분리해서 봄
- **"노드 CPU 사용률 (%)"** — `100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)`

1차 해결 이후 다시 부하테스트를 돌려서 이 패널들을 관찰함.

### 관찰된 증상

- CPU 스로틀링 패널: 순간적으로 파드당 최대 200까지 튐
- 노드 CPU 사용률 패널: 30~35%대 — 전혀 포화 안 됨
- `kubectl get hpa` 확인: `cpu: 1%/70%`, replica는 4/8 그대로 — KEDA가 스케일아웃을 **안 한 게 아니라 정상 판단으로 안 한 것**

### 원인 진단 — "스로틀링 ≠ 사용률"

이 세 지표를 같이 보면 얼핏 모순처럼 보이지만 사실 앞뒤가 맞음:

- KEDA(HPA)의 CPU 트리거는 **limit이 아니라 request(500m) 대비 평균 사용률**을 봄
- 순간적으로 limit(2 CPU)까지 튀었다가 금방 내려가는 버스트라면, 5분 평균으로는 request 대비 여전히 낮게 잡혀서 70% 문턱을 못 넘음
- 즉 지금 겪는 건 "파드 개수가 부족한 것"이 아니라 "**개별 파드가 순간적으로 자기 CPU limit에 부딪히는 것**" — 클러스터 전체(노드 CPU 30%대)로 보면 여유가 충분한데, 그 여유를 개별 파드가 limit 때문에 못 끌어다 쓰는 상황

### 판단 (권고 — 2026-08-13 기준 아직 미적용)

replica 수(KEDA `minReplicas`/`maxReplicas`)를 손대는 것보다, **개별 파드 CPU limit을 노드 여유에 맞춰 더 올리는 쪽**(예: 2 → 3~4)이 순간 스로틀링을 줄이는 데 더 직접적인 처방. KEDA의 `maxReplicas`는 나중에 평균 사용률이 실제로 70%에 근접하는 걸 관찰한 뒤에 늘려도 늦지 않음 — 지금 늘려봐야 트리거 조건(평균 사용률)을 못 넘으니 켜지지도 않음.

### 참고 — 프론트엔드 "튀는" 현상과의 연관 가능성

같은 날 프론트엔드에서 "화면이 튄다"는 리포트가 있었음. 프론트 SSR 로그에서 `TypeError: fetch failed` / `ECONNREFUSED`(백엔드 Service로)를 발견했으나, 타임스탬프 확인 결과 리포트 시점보다 약 3시간 전(KEDA/ScaledObject 최초 적용 시점과 겹침) 로그였고 재현 테스트(파드 내부에서 직접 fetch)는 `200 OK`로 정상이었음. 확정은 아니지만, 순간 CPU 스로틀링으로 인한 응답 지연·타임아웃이 그 순간의 "튀는" 것처럼 보이는 현상과 연결됐을 가능성이 있음 — 위 limit 상향이 이 증상 완화에도 도움이 될 수 있음.

## 관련
- [[keda-scaling-missing-metrics-server]]
- [[hikaricp-connection-storm-load-test]]
- [[frontend-cpu-throttling-cfs-quota-vs-jvm-tradeoff]]
- [[../architecture/keda-autoscaling]]
- [[../decisions/2026-08-18-capacity-planning-large-traffic-readiness]]
