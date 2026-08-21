---
title: 새 backend 파드가 한 노드에 몰려서 뜰 때 JVM 콜드스타트 CPU 경합 → liveness 재시작 / 헬스 붕괴
category: troubleshooting
tags: [keda, karpenter, jvm, cpu-throttling, rollout, load-test]
created: 2026-08-20
updated: 2026-08-21
---

> 🟢 2026-08-21: prod에서 실제로 재현됨(2000명 e2e 부하테스트 중, backend 8/8 스케일업 자체는
> 성공했으나 신규 파드 4개가 CPU 103%까지 찬 노드 하나에 몰려서 재시작 반복 — `FailedScheduling`
> → Karpenter 신규 노드 프로비저닝 → 그 노드에 몰려서 뜨는 패턴까지 동일하게 확인됨). 아래
> "재발 방지"의 `topologySpreadConstraints` 권고를 실제로 적용함 — `CD/helm/templates/
> backend-deployment.yaml`/`frontend-deployment.yaml` 둘 다 `maxSkew: 1`,
> `topologyKey: kubernetes.io/hostname`로 추가(`helm template` 렌더링 확인). CD 레포
> 커밋/배포는 사용자 몫 — 이 세션에서는 코드만 반영.
>
> 🔴 **같은 날 후속 — `ScheduleAnyway`(소프트)로는 이 문제가 그대로 재현됨.** 배포 후 돌린
> 다음 부하테스트에서 실제 스케줄링 이벤트를 확인해보니 `FailedScheduling: 0/2 nodes are
> available: 2 Insufficient cpu` — 스케일업 순간 기존 노드가 전부 이미 꽉 차있어서, Karpenter가
> 신규 파드 4개 전부를 수용할 새 노드를 "비용 최적화상 하나만" 만들어버림. `topologySpreadConstraints`는
> "이미 있는 노드들 사이" 분산 규칙이라 애초에 분산시킬 다른 노드 자체가 없었고,
> `ScheduleAnyway`라 이 상황(제약을 못 지키는 배치)을 그냥 허용해버림. `whenUnsatisfiable`을
> `DoNotSchedule`(하드 제약)로 변경 — Karpenter가 있어서 "제약 못 맞추면 Pending" 위험이 낮고,
> 대신 필요하면 노드를 여러 개로 나눠 만들어서라도 분산을 강제하게 됨.
>
> 🟡 **또 같은 날 후속 — `DoNotSchedule`도 완벽한 해법은 아니었음, 새로운 트레이드오프 발견(미해결).**
> `DoNotSchedule` 배포 후 재검증 테스트에서 로그인 대량 timeout(`request timeout`)이 실제로 터짐.
> 원인은 이전과 정반대 방향 — `DoNotSchedule`이 의도대로 "분산"을 강제하긴 했는데, 이번엔
> 스케일업에 신규 노드가 **5개나 동시에 필요**했고(각 파드가 서로 다른 노드를 요구하니까), 그
> 5개 노드가 전부 뜨는 데 1~2분이 걸리는 동안 이미 떠있던 파드 2개(목표 8개 중)가 트래픽을
> 전부 떠안아서 `HPA cpu 399%/70%`(목표의 5.7배)까지 과부하됨 → 그 2개 파드도 응답을 못
> 해서 로그인 대량 timeout으로 이어짐. 즉 `ScheduleAnyway`는 "몰리지만 빨리 준비됨",
> `DoNotSchedule`은 "고르게 분산되지만 신규 노드를 여러 개 기다려야 해서 늦게 준비됨" —
> 어느 쪽이 실제로 더 안전한지는 상황(한 번에 필요한 신규 노드 수, Karpenter 노드 부팅
> 속도)에 따라 다르다는 게 실측으로 드러남. **아직 최종 결론 안 냄** — 후보: (a) 그대로
> 유지하고 감수, (b) `ScheduleAnyway`로 롤백, (c) `maxSkew`를 2~3 정도로 완화해서 "1노드
> 4개 쏠림"은 막되 "1노드 2개"는 허용하는 절충안. 다음 세션에서 이어서 정할 것.
>
> 이 재검증 과정에서 부수적으로 두 가지 더 확인/정리함:
> - k6 스크립트(`Infra/loadtest/e2e_reservation_2000.js`)의 `reservation_unexpected_fail`
>   threshold가 이런 정상적인(비록 트레이드오프가 있더라도) 스케일업 blip까지 "버그"로 잡아내는
>   문제 — CloudWatch로 확인해보니 그 blip 동안 `HTTPCode_Target_5XX_Count`(앱이 직접 준 500)는
>   0이었고 `TargetConnectionErrorCount`(ALB가 파드에 연결 자체를 못 맺음)만 튀었음 — 앱 버그가
>   아니라 인프라 레벨 현상임을 확인. threshold를 `count==0` → `count<20`(2000명의 1%)으로 완화.
> - 부하테스트를 반복 실행하려면 매번 이전 실행이 남긴 "좀비 대기열 상태"(Redis
>   `queue:{roundId}:waiting`/`active`에 만료됐거나 안 지워진 토큰들)와 "이미 소진된 좌석"(DB
>   `RESERVATIONS.reserved_status`)을 초기화해야 함 — 절차는 [[../runbook/loadtest-round-reset]] 참고(신설).

# 새 backend 파드 동시 콜드스타트 CPU 경합 — 두 번 재현(KEDA 스케일업, 롤링배포)

2026-08-20 Gateway API 마이그레이션 직후 2000명 규모 e2e 부하테스트 도중, 같은 근본 원인이 두 가지 다른 트리거로 재현됨.

## 공통 메커니즘

1. 여러 backend 파드가 **같은 노드에, 거의 동시에** 뜸(스케줄링이 몰리는 건 KEDA 스케일업이든 Deployment 롤링배포든 흔히 생김)
2. JVM 콜드스타트(클래스 로딩, Spring 빈 초기화, HikariCP가 풀 사이즈만큼 커넥션을 한꺼번에 여는 것)는 짧은 순간 CPU를 많이 씀
3. CFS는 **100ms 단위**로 순간 사용량을 계산하는데, 파드 CPU limit이 넉넉해 보여도(예: 3코어) 같은 노드에서 여러 파드가 이 부트스트랩 버스트를 동시에 내면 노드 전체 코어 수를 순간적으로 초과 → 스로틀링
4. 스로틀링 때문에 앱이 `initialDelaySeconds`+`failureThreshold×periodSeconds` 유예시간(liveness 약 50초, readiness 약 25초) 안에 준비를 못 마침 → probe 실패 → 재시작 또는 "Not Ready" 처리

## 재현 1 — KEDA 스케일업 (자가치유됨)

부하테스트 시작 직후 KEDA가 backend replica를 늘리는데, 기존 노드에 여유가 없어서 Karpenter가 새 노드를 프로비저닝. 그 새 노드에 파드 3개가 몰려서 뜨며 liveness probe가 `connection refused`/`context deadline exceeded`로 실패 → kubelet이 재시작(`exitCode 143`, SIGTERM). **재시작 1회 후 정상화** — 타겟그룹도 계속 healthy 유지돼서 심각한 영향은 없었음.

## 재현 2 — 설정 변경으로 인한 전체 롤링배포 (실제 다운타임 발생)

[[hikaricp-pool-stale-sizing-after-rds-upgrade]]의 `dbPoolSize`/`maxReplicas` 수정이 pod template hash를 바꿔서 **backend 전체가 롤링 재배포**됨. 이번엔 여러 새 파드가 한 노드(`ip-10-70-2-213`)에 몰려서 뜨며 readiness probe가 대거 실패했고, `RollingUpdate`(`maxUnavailable: 25%, maxSurge: 25%`)가 보장하는 "최대 25%만 내려간다"는 한도를 실질적으로 넘어섬 — **이유: `maxUnavailable`은 "옛 파드를 몇 개까지 자발적으로 내릴지"만 제한하지, 새로 올라온(surge) 파드가 readiness에 실패해서 카운트 안 되는 상황까지는 막아주지 않는다.** 옛 파드는 스케줄대로 계속 내려가는데 새 파드가 준비가 안 되니, 실제 가용 용량이 급격히 줄어듦.

CloudWatch `HealthyHostCount`(대상 타겟그룹)로 직접 확인:
```
17:44~17:50: 4 (평상시)
17:51~17:58: 5→12 (KEDA 스케일업)
17:59~18:07: 12→4 (정상 스케일다운)
18:08: 1  ← 붕괴
18:09~18:16: 5→8 (회복)
```
이 `18:08`의 붕괴 순간과 겹친 부하테스트 요청들이 실제로 실패함 — 같은 시점 k6 결과에서 로그인 성공률이 57%(1158/2000)까지 떨어짐. 로그인 요청은 `RAMP_SECONDS`(기본 10초) 지터로 테스트 시작 직후 짧은 구간에 몰리는 구조라, 마침 이 1분짜리 붕괴 구간과 겹치면 상당수가 실패로 잡힌다.

## 재현 3 — `topologySpreadConstraints(DoNotSchedule)` 적용 후에도 다른 형태로 재현 (2026-08-21, prod)

위 두 재현을 겪고 `topologySpreadConstraints`(처음엔 `ScheduleAnyway`, 그다음 `DoNotSchedule`)를 실제로 적용했는데, `DoNotSchedule`로 바꾼 뒤 재검증 테스트에서 또 다른 모양으로 문제가 재현됨. 자세한 내용은 이 문서 맨 위 알림(🟡) 참고 — 핵심만 요약하면: "노드 하나에 몰리는 것"은 확실히 막혔지만, 그 대신 "스케일업에 필요한 신규 노드 여러 개가 동시에 뜨는 걸 기다리느라 스케일업 자체가 늦어져서, 그동안 기존 소수 파드가 과부하되는" 새로운 실패 모드가 나타남. `HPA cpu 399%/70%`, 로그인 대량 timeout으로 실제 확인됨. **아직 미해결 — 다음 세션에서 `maxSkew` 완화나 롤백 여부를 결정할 것.**

## 재발 방지 / 고려할 것 (미적용)

- **설정 변경으로 인한 전체 재배포는 "배포"로 취급하고, 부하테스트와 시간을 겹치지 않게 할 것** — 이번 사고의 직접 원인은 "부하테스트 도중 무심코 설정을 고쳐서 롤링배포가 트리거된 것" 자체였음.
- Deployment `strategy.rollingUpdate`에 `maxUnavailable: 0`을 고려할 수 있음(옛 파드를 새 파드가 완전히 Ready된 뒤에만 내리는 방식) — 다만 `maxSurge`만으로 여유 용량을 확보해야 해서 순간적으로 필요한 파드 수가 더 늘어남(비용/노드 여유 트레이드오프).
- `PodDisruptionBudget`으로 롤아웃 중에도 최소 가용 개수를 강제하는 것도 검토 가능.
- 근본적으로는 [[frontend-cpu-throttling-cfs-quota-vs-jvm-tradeoff]]에서 frontend에 적용했던 것처럼 **backend도 CPU limit을 없애고 request 기반 fair-share로 전환**하는 방법이 있으나, backend는 JVM이 `ActiveProcessorCount`를 cgroup 쿼터에서 자동으로 추론하는 구조라 limit을 없애면 스레드풀이 과다 산정될 위험이 있어(이미 문서화된 트레이드오프) 신중한 검토 필요.
- 🟡 (부분 적용, 2026-08-21 — 새 트레이드오프 발견, 미해결) **파드 분산 강제**(topologySpreadConstraints/anti-affinity)로 같은 배포 이벤트에서 여러 신규 파드가 같은 노드에 몰리지 않게 하는 것 — "재현 3"과 맨 위 알림 참고. 몰림은 막혔지만 신규 노드 여러 개를 기다리느라 스케일업이 늦어지는 부작용이 새로 생김.

## 관련
- [[hikaricp-pool-stale-sizing-after-rds-upgrade]]
- [[backend-cpu-throttling-and-scaling-load-test]]
- [[frontend-cpu-throttling-cfs-quota-vs-jvm-tradeoff]]
- [[../decisions/2026-08-18-capacity-planning-large-traffic-readiness]]
- [[argocd-besteffort-and-bootstrap-node-capacity]]
- [[admin-alb-malformed-cidr-fixed-response-500]]
- [[../runbook/loadtest-round-reset]]
