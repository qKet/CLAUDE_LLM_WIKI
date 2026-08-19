---
title: 평균 응답시간 패널이 트래픽 뜸한 엔드포인트에서 터무니없이 큰 값(20013ms)을 보여주던 문제
category: troubleshooting
tags: [monitoring, grafana, prometheus, promql]
created: 2026-08-19
updated: 2026-08-19
---

# 평균 응답시간 패널이 트래픽 뜸한 엔드포인트에서 터무니없이 큰 값을 보여주던 문제

## 증상

Grafana `qket` 대시보드의 "평균 응답시간 (ms, 엔드포인트별)" 패널에서 `/auth/login` 타일이 **20013**(ms)로 표시됨. 로그인이 20초씩 걸린다는 뜻인데, 실제로는 그렇지 않았음(같은 시간대 다른 검증에서 정상 확인).

## 원인

패널 쿼리:
```
sum by (uri) (rate(http_server_requests_seconds_sum{...}[$__rate_interval]))
/ sum by (uri) (rate(http_server_requests_seconds_count{...}[$__rate_interval])) * 1000
```

공식 자체는 맞음(합계/건수 = 평균). 문제는 두 가지가 겹침:

1. `/auth/login`은 부하테스트할 때만 몰렸다가 끊기는 **매우 버스트성** 트래픽 — 대부분 시간엔 요청이 거의 0건.
2. 요청이 거의 없는(표본이 희박한) 구간에서 `rate()`는 수치적으로 불안정해짐 — 분모(count의 rate)가 0에 가까워지면 나눗셈 결과가 튀거나(이번 경우처럼) `0/0 = NaN`이 됨.

패널의 `reduceOptions.calcs: ["lastNotNull"]` 설정 때문에, 지금 이 순간 값이 NaN이어도 **선택된 시간범위 안에서 마지막으로 유효했던(NaN이 아닌) 값**을 계속 보여줌 — 그게 트래픽 경계 구간에서 튄 비정상적인 숫자였음. 실제로 그 순간 재계산해보면 `nan`이 나오는 것을 직접 쿼리로 확인함.

## 해결

`02_k8s-addon/dashboards/qket-monitoring.json`의 해당 쿼리에 요청률 하한 필터 추가:

```diff
- sum by (uri) (rate(http_server_requests_seconds_sum{...}[$__rate_interval]))
- / sum by (uri) (rate(http_server_requests_seconds_count{...}[$__rate_interval])) * 1000
+ (sum by (uri) (rate(http_server_requests_seconds_sum{...}[$__rate_interval]))
+ / sum by (uri) (rate(http_server_requests_seconds_count{...}[$__rate_interval])) * 1000)
+ and (sum by (uri) (rate(http_server_requests_seconds_count{...}[$__rate_interval])) > 0.01)
```

PromQL `and`로, 요청률이 사실상 0에 가까운(표본이 불안정한) 시점의 값은 아예 결과에서 제외 — 그 시점엔 "No data"로 보이는 게 맞고(실제로 최근 트래픽이 없으니까), 트래픽이 있을 때만 신뢰할 수 있는 평균이 표시됨.

## 재발 방지

- **`rate()`/`increase()` 기반 "평균"류 stat 패널은 트래픽이 뜸하거나 버스트성인 지표에서 항상 이 문제를 겪을 수 있음** — 요청량이 안정적인 엔드포인트(`/categories`, `/events/paged` 등)에서는 문제없이 정상 값이 나왔던 것도 이 가설을 뒷받침함.
- 이런 stat 패널을 새로 만들 때는 요청률이 낮은/버스트성인 지표에 대해 `and (rate(count[...]) > threshold)` 같은 하한 필터를 기본으로 고려할 것.
- 비슷한 계열 문제로 [[grafana-amp-rate-interval-sparse-historical-data]](같은 `rate()`/`increase()` 고정 윈도우 함정, 원인은 다르지만 "간헐적 트래픽 + rate() 수학"이라는 공통 패턴)가 있었음.

## 관련
- [[grafana-amp-rate-interval-sparse-historical-data]]
- [[servicemonitor-actuator-port-mismatch]] — 같은 대시보드를 고치던 중 연쇄로 발견됨
