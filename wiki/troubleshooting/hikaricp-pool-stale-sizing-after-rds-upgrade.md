---
title: RDS 인스턴스 업그레이드 후 재계산 안 한 HikariCP 풀 사이징 — 2000명 e2e 테스트에서 실제 고갈
category: troubleshooting
tags: [hikaricp, rds, keda, capacity, load-test]
created: 2026-08-20
updated: 2026-08-20
---

# RDS 업그레이드 후 재계산 안 한 HikariCP 풀 사이징 — 실제 고갈 확인

Gateway API 마이그레이션 직후 진행한 2000명 규모 e2e 부하테스트(`Infra/loadtest/e2e_reservation_2000.js`) 중 발견.

## 증상

```
HikariPool-1 - Connection is not available, request timed out after 3000ms
(total=10, active=10, idle=0, waiting=15~37)
```
10분간 68건 발생. `SeatMapper.findByRoundId` 등 실제 쿼리가 커넥션을 못 받아 500으로 응답.

## 원인 — 계산은 맞았는데, 기준이 된 인스턴스 클래스가 낡음

`CD/helm/values.yaml`에 이미 있던 규칙:
```
dbPoolSize × maxReplicas ≤ RDS max_connections
```
2026-08-13에 이 계산을 도입할 당시 기준은 `db.t3.micro`(max_connections ~85)였고, 그래서 `dbPoolSize=10, maxReplicas=8` (80, 여유 있게)로 잡아뒀었다.

근데 2026-08-18 대용량 트래픽 대비 분석([[../decisions/2026-08-18-capacity-planning-large-traffic-readiness]])에서 RDS가 `db.t3.medium`으로 업그레이드됐는데, **이 계산식을 다시 안 돌렸다.** `maxReplicas`가 그 사이 8→12로 늘면서 `10×12=120`이 됐는데, 이건 `db.t3.micro` 기준으로는 이미 초과값(85 넘음)이었을 텐데도 몰랐던 이유는 — RDS가 이미 `db.t3.medium`으로 바뀌어 있어서 표면적인 장애 없이 그냥 지나갔던 것.

실제 `db.t3.medium`의 진짜 `max_connections`를 RDS 파라미터그룹에서 직접 확인:
```bash
aws rds describe-db-parameters --db-parameter-group-name default.mysql8.0 \
  --query "Parameters[?ParameterName=='max_connections']"
# ParameterValue: "{DBInstanceClassMemory/12582880}" (RDS 기본 공식, db.t3.medium 4GB 기준 ≈ 341)
```
즉 **실제로는 341까지 여유가 있는데 120에서 인위적으로 막혀있었던 것.** 부하테스트 중 CloudWatch `DatabaseConnections`가 정확히 120에서 5분 넘게 고정된 것으로 확인.

## 해결

`CD/helm/values.yaml`:
```yaml
dbPoolSize: 10 → 12
autoscaling.maxReplicas: 12 → 20
```
`12×20=240`, 341의 약 70% — 여유를 남겨둠. 노드 CPU도 8~19%로 여유로워서 `maxReplicas` 상향 자체는 문제없음(Karpenter가 필요하면 노드도 늘려줌).

## 재발 방지

- **RDS 인스턴스 클래스를 바꿀 때마다 `dbPoolSize × maxReplicas` 재계산을 체크리스트에 넣을 것** — 이번처럼 "값을 안 바꿔도 장애가 안 나서 놓치는" 패턴이 제일 위험하다(용량이 커진 방향이라 당장은 문제가 없었고, 부하가 실제로 커져야만 드러남).
- `aws rds describe-db-parameters ... max_connections`로 실제 한도를 확인하는 걸 습관화 — RDS 공식(`DBInstanceClassMemory/12582880`)은 인스턴스 클래스별로 자동 계산되므로 문서 상 숫자만 보고 짐작하지 말 것.
- 이 설정 변경 자체가 pod template hash를 바꿔서 **전체 backend가 롤링 재배포**된다 — 그 파급 효과는 [[backend-cold-start-cpu-contention-during-rollout]] 참고(같은 세션에서 바로 재현됨).

## 관련
- [[../decisions/2026-08-18-capacity-planning-large-traffic-readiness]]
- [[hikaricp-connection-storm-load-test]] — 반대 방향(풀이 너무 커서 생긴) 사고, 이번 건과 대조됨
- [[backend-cold-start-cpu-contention-during-rollout]]
