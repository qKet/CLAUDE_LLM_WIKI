---
title: AMP 데이터소스로 넓은 시간 범위 조회 시 예전 데이터가 빈 값(No data)으로 보임
category: troubleshooting
tags: [monitoring, grafana, amp, prometheus, promql]
created: 2026-08-18
updated: 2026-08-18
---

# AMP 데이터소스로 넓은 시간 범위 조회 시 예전 데이터가 빈 값(No data)으로 보임

## 증상

[[grafana-amp-datasource-missing-auth-token]]를 해결하고 대시보드 패널을 전부 AMP 데이터소스로 전환한 뒤, 시간 범위를 넓게(예: "Last 7 days") 잡으면 최근 몇 시간은 그래프가 나오는데 그보다 예전 구간(예: 2026-08-13)은 대부분 빈 값(No data / 그래프가 뚝 끊긴 것처럼 보임)으로 표시됨.

## 원인

대시보드 패널의 PromQL 쿼리가 `rate(...[5m])`처럼 **고정된 5분 lookback 윈도우**를 쓰고 있었음. `rate()`/`increase()` 같은 함수는 "각 쿼리 시점 기준으로 그 직전 구간(`[5m]`)에 실제 샘플이 있어야만" 값을 계산할 수 있음.

`01_infrastructure`가 매일 밤 destroy/재생성되는 구조라, EKS가 꺼져있던 시간대엔 Prometheus 자체가 없어서 당연히 샘플이 없음 — 그래서 예전 데이터(예: 2026-08-13 부하테스트 당일)는 클러스터가 켜져있던 짧은 시간대에만 듬성듬성 찍혀있음.

넓은 시간 범위(7일)로 그래프를 그릴 때 Grafana는 데이터 포인트 개수를 적당히 유지하려고 큰 간격(예: 30분~1시간)으로 쿼리 시점을 잡는데, 그 간격이 `[5m]` 윈도우보다 훨씬 크면 **대부분의 쿼리 시점이 "직전 5분 안에 샘플이 있는" 순간을 놓쳐서** 빈 값이 됨. 반면 지금(실시간) 구간은 Prometheus가 계속 데이터를 넣고 있어서 어느 시점에 쿼리해도 항상 "직전 5분 안"에 샘플이 있어 정상적으로 보였음 — 그래서 마치 "최근 데이터만 있고 예전 건 사라진 것"처럼 보였음.

## 해결

`rate()`/`increase()`를 쓰는 쿼리의 `[5m]`을 Grafana 내장 변수 **`$__rate_interval`**로 교체.

```diff
- sum(rate(container_cpu_usage_seconds_total{...}[5m])) * 100
+ sum(rate(container_cpu_usage_seconds_total{...}[$__rate_interval])) * 100
```

`$__rate_interval`은 지금 보고 있는 시간 범위/해상도(스크래핑 주기 포함)에 맞춰 **윈도우 크기를 자동으로 늘려주는** 변수라, 넓게 볼 때는 윈도우도 같이 넓어져서 듬성듬성한 과거 샘플도 제대로 잡아냄. 좁게 볼 때는 윈도우도 좁아져서 최근 데이터 해상도가 나빠지지 않음.

`02_k8s-addon/dashboards/qket-monitoring.json`에서 `rate()`/`increase()` 쓰는 9곳 전부 교체 → 7일 범위 조회 시 데이터 포인트 전부(10/10) 값이 채워지는 것 확인함(수정 전엔 대부분 비어있었음).

## 재발 방지

- **Prometheus 대시보드에서 `rate()`/`increase()`를 쓸 땐 `[5m]` 같은 고정값 대신 `$__rate_interval`(또는 최소한 `$__interval`)을 기본으로 쓸 것** — 특히 이 프로젝트처럼 클러스터가 간헐적으로만 켜져있어서 데이터가 듬성듬성한 환경에서는 고정 윈도우가 예전 데이터를 조용히 숨겨버리는 문제가 생기기 쉬움.
- 새 패널 만들 때 반드시 넓은 시간 범위(예: 7일, 30일)로도 한 번씩 확인해볼 것 — 기본 시간 범위(6시간)만 보고 넘어가면 이런 문제를 못 알아챔.

## 관련
- [[grafana-amp-datasource-missing-auth-token]] — 이 문제를 발견하게 된 전제 조건(AMP 조회 자체가 이때 처음 가능해짐)
