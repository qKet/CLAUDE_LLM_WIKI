---
title: 만 명 오픈런 부하테스트 — k6 클라이언트 함정 3가지 + ALB 헬스체크 연쇄 장애
category: troubleshooting
tags: [load-test, k6, keda, autoscaling, alb, health-check, cascading-failure, argocd]
created: 2026-08-18
updated: 2026-08-18
---

# 만 명 오픈런 부하테스트 — k6 클라이언트 함정 3가지 + ALB 헬스체크 연쇄 장애

[[../decisions/2026-08-18-capacity-planning-large-traffic-readiness]]의 4번 권고("목표 동접자 수로 재측정")를 실제로 실행하면서 겪은 문제들. `Infra/loadtest/open_run_10000_no_queue.js`/`open_run_10000_with_queue.js`로 500명 → 만 명까지 단계적으로 올리면서, 순서대로 서로 다른 층위의 문제를 4개 발견함(k6 클라이언트 3개, 서버 인프라 1개 — 이게 제일 심각).

## 문제 1 — `constant-vus`가 VU 수를 부풀림 (500명 테스트에서 발견)

### 증상
`options.scenarios`에 `executor: 'constant-vus'`, `duration: '2m'`만 주고 `iterations`를 안 줬더니, 500 VU로 돌렸는데 결과에 `iterations: 4742`가 찍힘.

### 원인
`constant-vus`는 각 VU가 `duration` 동안 시나리오를 **계속 반복**하는 executor. 500명이 "한 번씩" 시도한 게 아니라 500명이 2분간 평균 9번씩 재시도해서, 로그인만 초당 ~40건(수천 명이 몰린 것과 비슷한 부하)이 나갔음. Grafana에서 본 노드 CPU 100% 스파이크도 상당 부분 이 "가짜 규모"였을 가능성이 큼.

### 해결
`executor: 'per-vu-iterations'` + `iterations: 1`로 변경 — VU 수(`VUS`)가 정확히 "그 순간 시도한 사람 수"와 일치하게 됨. `duration` 옵션도 `maxDuration`(반복 시간이 아니라 전체 종료 안전 상한선)으로 의미가 바뀜.

## 문제 2 — macOS에서 VU 수천 개 = k6 자체 크래시

### 증상
```
crypto/x509.(*Certificate).systemVerify
crypto/x509/internal/macos.SecTrustEvaluateWithError
runtime: program exceeds 10000-thread limit
```
같은 goroutine 스택트레이스와 함께 k6 프로세스 자체가 죽음.

### 원인
macOS는 TLS 인증서 검증을 Go 코드가 아니라 시스템(Security.framework)에 **블로킹 syscall**로 위임함. VU 수천 개가 거의 동시에 HTTPS 커넥션을 열면 그만큼 OS 스레드가 한꺼번에 이 블로킹 호출에 걸리고, Go 런타임의 스레드 상한(1만 개)을 넘겨서 런타임이 안전장치로 프로세스를 죽여버림. **서버(EKS) 문제가 아니라 로컬 테스트 클라이언트의 OS 레벨 한계.**

### 해결
`options.insecureSkipTLSVerify: true` — 우리 도메인이라 인증서 검증 자체가 불필요해서 이 무거운 경로를 회피. (근본적으로 회피하려면 Linux에서 돌리면 됨 — 이 syscall 문제 자체가 없음.)

## 문제 3 — VU 1만 개 동시 커넥션 = ALB TLS negotiation 실패

### 증상
클라이언트 쪽 macOS 크래시(문제 2)를 고쳤는데도 계속 `connection reset by peer` 발생.

### 원인 진단
CloudWatch `AWS/ApplicationELB` 지표를 직접 조회:
```
ClientTLSNegotiationErrorCount:  분당 6천~8천 건
RejectedConnectionCount:         0
TargetConnectionErrorCount:      0
```
backend/frontend가 아니라 **ALB 자체가 TLS 핸드셰이크 단계에서 리셋**시키고 있었음. ALB는 트래픽 추세를 보고 자기 용량(LCU)을 점진적으로 늘리는 구조라, `per-vu-iterations`가 만든 "진짜로 같은 순간"의 폭증은 AWS 인프라 레벨에서도 못 받아냄 — AWS 공식 문서도 대규모 스파이크 예정 시 사전 pre-warming을 권장할 정도.

### 해결
각 VU가 요청을 시작하기 전 `0~RAMP_SECONDS`초(기본 10초) 사이에서 무작위로 대기하게 함(`sleep(Math.random() * RAMP_SECONDS)`). "만 명이 각자 한 번씩"이라는 본질은 유지하면서 커넥션 개설 시점만 몇 초에 걸쳐 자연스럽게 분산 — 실제 오픈런도 완전히 같은 밀리초에 동시 접속하진 않으므로 오히려 더 현실적인 모델이 됨.

## 문제 4 — 🔴 진짜 서버 문제: ALB 헬스체크 연쇄 장애 (전체 다운, 미해결)

이 문서에서 가장 심각한 발견. 위 3개를 다 걷어내고 진짜 만 명 부하를 걸었더니 발생함.

### 증상
```
✗ 홈 200       10%  (1073 / 9927)
✗ 로그인 200   66%  (6612 / 3388)
✗ 상세 200     45%
✗ 좌석 조회 200 74%
```
가장 먼저·가장 많이 몰리는 "홈"이 압도적으로 나쁨(10%) — 다운스트림 단계보다 훨씬 나쁜 게 이상해서 추가 조사함.

### 원인 진단

CloudWatch로 타겟그룹 헬스 상태를 분단위로 조회해서 결정적 증거를 찾음:
```
20:17  backend  targetgroup  HealthyHostCount = 0,  UnhealthyHostCount = 12   ← 전멸
20:17  frontend targetgroup  HealthyHostCount = 0,  UnhealthyHostCount = 1    ← 전멸
```
같은 시각 `HTTPCode_ELB_5XX_Count`가 분당 최대 13383건까지 튀었는데, `HTTPCode_Target_5XX_Count`(타겟이 직접 준 5xx)는 그 10분의 1도 안 됨 — 즉 타겟이 에러를 준 게 아니라 **ALB가 라우팅할 healthy 타겟을 아예 못 찾아서** 자체적으로 5xx를 반환한 것.

타겟그룹 헬스체크 설정 확인:
```
간격: 15초, 타임아웃: 5초, unhealthy 판정: 2번 연속 실패(=30초)
backend 헬스체크 경로: /api/actuator/health
frontend 헬스체크 경로: /   ← 일반 트래픽과 완전히 같은 무거운 SSR 페이지
```

**연쇄 반응(cascading failure) 메커니즘**:
1. 요청 폭주 → 파드 CPU 포화, 요청 처리 지연
2. 그 와중에 **헬스체크도** 5초 안에 응답 못 함(특히 frontend는 헬스체크 경로가 일반 요청과 동일해서 제일 먼저 걸림)
3. 30초 연속 실패 → ALB가 해당 파드를 unhealthy 판정, 로테이션에서 제외
4. 남은 파드가 더 적은 인원으로 더 많은 트래픽을 받음 → 걔들도 곧 타임아웃 → 또 제외
5. 도미노로 전부 빠지면 **HealthyHostCount = 0** → 그 순간 요청 100% 실패
6. 부하가 잠시 빠지면 복귀 → 반복(20:11, 20:12, 20:17, 20:18에 반복적으로 재현됨)

즉 이 시스템은 과부하 상황에서 **"느리게라도 버티는" 게 아니라, 과부하 파드를 스스로 제거하는 안전장치가 오히려 완전 장애를 만드는 구조**임. 여유 파드가 없으니 제외당한 트래픽을 받아줄 곳이 없음.

### 부수적으로 같이 발견됨 — ArgoCD 자신도 이 부하에 희생됨

같은 시간대 ArgoCD UI에서 `connection error: dial tcp ...:8081: connect: connection refused` 발생. 확인해보니:
```
argocd-repo-server resources: {}   ← CPU/메모리 request/limit이 아예 없음 (QoS: BestEffort)

Events:
  Liveness probe failed: connect: connection refused (여러 번)
  Killing: failed liveness probe, will be restarted (3번 재시작됨)
```
노드 CPU가 부하테스트로 90%대까지 눌린 동안, request가 없는 BestEffort 파드가 커널 스케줄링에서 제일 먼저 밀려나서 자기 헬스체크(`:8084/healthz`)에도 제때 응답 못 하고 kubelet한테 강제 재시작당함. ArgoCD 컴포넌트 전반에 최소 리소스 request가 없다는 게 근본 원인 — **아직 미적용**.

### 재발 방지 (전부 미적용 — 다음 세션에서 이어서)

1. **frontend 헬스체크 경로 분리** — `/`(무거운 SSR) 대신 부하와 무관하게 즉답 가능한 전용 엔드포인트(예: `/api/health`, 정적 200 응답)로 타겟그룹 헬스체크 경로 변경. 이게 되면 트래픽이 몰려도 헬스체크만큼은 계속 통과해서 연쇄 제외 자체가 안 일어남 — 가장 우선순위 높은 조치
2. ArgoCD 컴포넌트(특히 repo-server)에 최소 CPU/메모리 request 부여 — `02_k8s-addon/main.tf`의 `helm_release.argocd` values에 `repoServer.resources` 추가
3. 헬스체크 자체를 더 관대하게(timeout 5→10초, unhealthy threshold 2→더 크게) 튜닝 — 진짜 장애 감지가 느려지는 트레이드오프 있음, 1번이 우선
4. 근본적으로는 [[../decisions/2026-08-10-redis-session-queue-shared-instance-risk]]와 연결되는 얘기 — 대기열(`open_run_10000_with_queue.js`)을 쓰면 유입 자체가 눌려서 이 연쇄 장애 자체가 원천 봉쇄됨. `no_queue` 버전으로 일부러 대기열 없이 테스트해서 이 문제가 고스란히 드러난 것

## 관련
- [[../decisions/2026-08-18-capacity-planning-large-traffic-readiness]]
- [[../architecture/cluster-autoscaler]]
- [[../architecture/keda-autoscaling]]
- [[frontend-cpu-throttling-cfs-quota-vs-jvm-tradeoff]]
- [[../decisions/2026-08-10-redis-session-queue-shared-instance-risk]]
