---
title: frontend CPU 스로틀링 — CFS 쿼터 구조 원인 규명, backend와 다르게 limit을 아예 없앤 이유
category: troubleshooting
tags: [monitoring, cpu-throttling, keda, cpu-limit, nodejs, jvm, grafana, prometheus]
created: 2026-08-18
updated: 2026-08-18
---

# frontend CPU 스로틀링 — CFS 쿼터 구조 원인 규명, backend와 다르게 limit을 아예 없앤 이유

## 증상

`frontend`에도 KEDA를 붙이면서(아래 관련 커밋: `c0da8a1 Feature: frontend keda 세팅`) Grafana "CPU 스로틀링(5분간, pod별)" 패널을 보니:

1. 롤아웃 순간(신규 ReplicaSet `67d96b7d69`이 구 `7b466c66b8`를 대체하는 구간) 스로틀링 값이 파드당 6~14까지 튐
2. 롤아웃이 끝나고 신규 파드만 남은 뒤에도, `kubectl top`으로는 CPU 사용량이 3~6m(1코어 limit의 0.3~0.6%, 사실상 유휴)인데 스로틀링 카운트가 **0이 아님**(파드당 5분에 1~3회)

1번은 롤링 업데이트 중 신/구 파드가 잠깐 겹쳐서 생기는 정상적인 일시 현상이라 넘어갔지만, 2번("평균 사용량은 거의 0인데 왜 스로틀링이 걸리지?")이 진짜 조사 대상이었음.

## 원인 진단

패널이 쓰는 쿼리(`02_k8s-addon/dashboards/qket-monitoring.json`)를 그대로 Prometheus에 직접 질의해서 실측:

```
sum(increase(container_cpu_cfs_throttled_periods_total{namespace="qket-release", container=~"qket-backend|qket-frontend"}[5m])) by (pod)
```

```
qket-backend-*  (4개 파드) → 전부 정확히 0
qket-frontend-* (4개 파드) → 1.06 / 3.20 / 2.12 / 1.08   ← 0이 아님
```

**"평균이 낮으면 스로틀링도 없다"는 가정 자체가 틀렸다는 게 실측으로 확인됨.** 이유는 CFS(Completely Fair Scheduler) 쿼터가 **100ms 단위 회계 윈도우**로 동작하기 때문:

- Next.js(Node.js) SSR 렌더링 자체는 메인 스레드 하나지만, GC, libuv 스레드풀(비동기 I/O), V8 백그라운드 JIT 컴파일러 같은 보조 스레드가 **같은 컨테이너 cgroup CPU 쿼터를 같이 공유**함
- 이 보조 스레드들이 짧은 순간(하나의 100ms 윈도우 안)에 우연히 겹치면, 그 순간만 limit(당시 1코어)을 넘겨서 스로틀링되고 즉시 회복됨
- 이 순간적인 초과는 1분/5분 단위 평균 사용률(`kubectl top`, KEDA의 CPU 트리거가 보는 지표)에는 거의 안 잡힘 — 그래서 "평균은 유휴인데 스로틀링은 있다"는, 얼핏 모순으로 보이는 상황이 실제로 성립함

즉 `limit`을 아무리 올려도(0.5→1로 이미 한 번 상향했었음) **근본적으로 같은 종류의 문제가 계속 재현될 수 있는 구조** — limit이 있는 한 100ms 윈도우 경합은 확률적으로 언제든 다시 걸림.

## 해결

`frontend.resources`에서 **CPU `limit`을 아예 제거**하고 `request`(0.5코어)만 남김 — `CD/helm/values.yaml`, `CD/helm/values-prod.yaml` 둘 다 반영. `memory` limit은 OOM 격리를 위해 그대로 유지.

```yaml
resources:
  requests:
    cpu: "0.5"
    memory: "128Mi"
  limits:
    memory: "256Mi"   # cpu 없음
```

CPU limit이 없으면 컨테이너는 CFS **하드 쿼터** 대신 request 기반 **cgroup cpu.shares(fair-share)** 로 스케줄링됨 — 노드에 여유가 있는 한 100ms 윈도우 경합 자체가 스로틀링으로 이어지지 않음. request는 스케줄링 보장값이자 KEDA `cpu` 트리거의 Utilization 계산 기준이라 그대로 둠(KEDA 동작에 영향 없음 — [[../architecture/keda-autoscaling]] 참고).

## 왜 backend는 그대로 뒀는가 (limit 유지)

같은 논리를 backend(JVM/Spring Boot)에 그대로 적용하면 안 됨 — Node.js와 JVM은 컨테이너 CPU 인지 방식이 근본적으로 다름:

- **Node.js**: libuv 스레드풀 크기(`UV_THREADPOOL_SIZE`, 기본 4)가 고정값이고 cgroup 쿼터를 보고 스스로 늘어나지 않음 → limit을 없애도 부작용이 없음
- **JVM**: Java 10+ 컨테이너 지원(`UseContainerSupport`, 기본 on)이 **cgroup CPU 쿼터(=limit)를 읽어서** `Runtime.availableProcessors()`를 결정하고, 이 값으로 GC 스레드 수·`ForkJoinPool.commonPool()` 병렬도·Tomcat/Undertow 기본 워커 스레드 수를 자동 사이징함

backend Dockerfile/Helm 어디에도 `-XX:ActiveProcessorCount` 같은 명시적 오버라이드가 없어서, JVM은 순전히 cgroup limit(현재 2코어)을 보고 스레드 수를 정하고 있음. 여기서 limit을 없애면 JVM이 **노드 전체 vCPU(t3.xlarge=4코어)** 기준으로 스레드를 그만큼 더 만들어버리는데, 실제로 스케줄링 보장되는 건 request(500m)뿐이라 "JVM이 쓸 수 있다고 착각하는 스레드 수"와 "실제 보장된 CPU"의 괴리가 커져서 컨텍스트 스위칭만 늘고 오히려 성능이 나빠질 위험이 있음.

게다가 backend는 실측(위 쿼리)상 이미 스로틀링이 **정확히 0** — [[backend-cpu-throttling-and-scaling-load-test]]에서 2026-08-13에 250m/1 → 500m/2로 올린 조치가 지금은 (유휴~저부하 기준) 잘 먹히고 있다는 뜻이라, 지금 시점엔 굳이 바꿀 이유가 없음. 단, 이건 유휴/저부하 기준 실측이라 실제 부하테스트 결과와는 다를 수 있음(아래 재발 방지 참고).

향후 backend에 더 여유가 필요해지면(오픈런 등 순간 버스트 대응):
1. limit을 없애기보다 **그냥 더 올리는 쪽**(예: 2 → 3~4코어)이 안전
2. 정말 limit을 없애고 싶다면 반드시 `-XX:ActiveProcessorCount=<request 기준 코어수>` 같은 JVM 플래그로 스레드 사이징을 직접 고정해야 부작용이 없음

## 재발 방지

- **단일 프로세스/이벤트루프 기반 워크로드(Node.js, Go 등 스레드 자동 확장을 안 하는 런타임)** → CPU limit 자체를 안 거는 게 기본값으로 안전. 대신 `request`를 실제 필요치에 맞게 잡고 노드 오토스케일링/모니터링으로 노드 레벨 과밀만 감시
- **컨테이너 인지형 런타임(JVM, .NET 등 cgroup 쿼터로 스스로 스레드/GC를 사이징하는 런타임)** → limit을 유지하거나, 없앨 거면 반드시 스레드 사이징 관련 런타임 플래그를 request 기준으로 명시 고정
- 새 서비스를 추가할 때 이 판단 기준(런타임이 cgroup 쿼터를 읽어 스스로 사이징하는가?)을 먼저 확인하고 CPU limit 정책을 정할 것

## 관련
- [[backend-cpu-throttling-and-scaling-load-test]]
- [[../architecture/keda-autoscaling]]
- [[../decisions/2026-08-18-capacity-planning-large-traffic-readiness]]
