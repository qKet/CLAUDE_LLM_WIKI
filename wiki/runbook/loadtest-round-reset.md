---
title: 부하테스트 반복 실행 전 회차(round) 상태 초기화 절차
category: runbook
tags: [load-test, k6, redis, mysql, prod]
created: 2026-08-21
updated: 2026-08-21
---

# 부하테스트 반복 실행 전 회차(round) 상태 초기화 절차

## 배경

`Infra/loadtest/e2e_reservation_2000.js`(로그인→대기열→좌석선택→예매) 같은 e2e 부하테스트를 실제 회차(`ROUND_ID`)로 여러 번 반복 실행하면, 이전 실행이 남긴 두 가지 잔여 상태가 다음 실행을 왜곡시킨다:

1. **DB(`RESERVATIONS`)** — 좌석이 이미 `RESERVED`로 채워져서, 다음 테스트는 "2000명이 2000석에 도전"이 아니라 "2000명이 몇 석 안 남은 좌석에 도전"하는 다른 시나리오가 됨.
2. **Redis 대기열(`queue:{roundId}:waiting`/`active`, `queue:token:*`, `queue:user:{roundId}:*`)** — 만료됐거나 처리 안 된 좀비 토큰이 그대로 남아서, 새로 들어오는 사용자가 이 좀비들 뒤에 줄을 서게 되거나(대기 순번 왜곡) `MAX_ACTIVE_USERS`가 실제로는 비었는데 꽉 찬 것처럼 착각하게 만듦(단, `removeExpiredActive()`가 다음 admission 체크 때 만료분은 자동으로 지워주므로 `active`는 방치해도 자가치유되지만, `waiting`은 그런 자동 정리가 없음).

이 절차는 이 두 상태를 안전하게 초기화하는 방법을 정리한 것. **실행 전 반드시 "지우려는 예약이 전부 `loadtest%` 계정 것인지" 확인할 것** — round_id를 실제 서비스 고객도 쓰는 살아있는 회차로 잡았다면 진짜 고객 예매가 섞여있을 수 있음.

## 절차

### 0. 사전 확인 (진짜 고객 예매가 섞여있지 않은지)

```sql
SELECT
  CASE WHEN user_id LIKE 'loadtest%' THEN 'loadtest' ELSE 'real_or_null' END AS kind,
  COUNT(*) AS cnt
FROM RESERVATIONS
WHERE round_id = <ROUND_ID> AND reserved_status = 'RESERVED'
GROUP BY kind;
```
`real_or_null`이 0이 아니면 절대 진행하지 말 것 — 실제 고객 예매를 건드리게 됨.

### 1. Redis 대기열 상태 삭제

클러스터 내부에서 임시 pod로 접속(로컬에 redis-cli 없어도 됨):
```bash
REDIS_HOST=$(cd Infra/04_data/prod && terraform output -raw redis_endpoint)  # 또는 dev-redis(release)
kubectl run redis-cleanup --restart=Never --image=redis:7-alpine -n <namespace> --command -- sleep 120
kubectl wait --for=condition=Ready pod/redis-cleanup -n <namespace> --timeout=60s
kubectl exec -n <namespace> redis-cleanup -- sh -c "redis-cli -h $REDIS_HOST KEYS 'queue:*' | xargs redis-cli -h $REDIS_HOST DEL"
kubectl delete pod redis-cleanup -n <namespace> --wait=false
```
`queue:*` 패턴만 지우므로 `spring:session:sessions:*`(실제 로그인 세션)는 안 건드림 — 삭제 전후로 `KEYS "*"`를 한 번 찍어서 의도한 패턴만 지워졌는지 확인하는 걸 권장.

### 2. DB 좌석 상태 리셋

역시 임시 pod로 RDS에 직접 접속(로컬 mysql 클라이언트가 `mysql_native_password` 플러그인 문제로 안 될 수 있음 — 그럴 땐 클러스터 안에서 `mysql:8.0` 이미지로 접속):
```bash
kubectl run mysql-cleanup --restart=Never --image=mysql:8.0 -n <namespace> --command -- sleep 120
kubectl wait --for=condition=Ready pod/mysql-cleanup -n <namespace> --timeout=60s
kubectl exec -n <namespace> mysql-cleanup -- mysql -h <rds-endpoint> -uadmin -p"<password>" -D qket -e "
UPDATE RESERVATIONS
SET user_id = NULL, reserved_status = 'AVAILABLE', reserved_at = NULL, upt_id='SYSTEM'
WHERE round_id = <ROUND_ID> AND reserved_status = 'RESERVED' AND user_id LIKE 'loadtest%';

DELETE FROM RESERVATION_HISTORY WHERE round_id = <ROUND_ID> AND user_id LIKE 'loadtest%';
"
kubectl delete pod mysql-cleanup -n <namespace> --wait=false
```
좌석 행 자체는 `DELETE`하지 않고 `AVAILABLE`로 되돌림 — `RESERVATIONS`는 "좌석-회차 슬롯"을 나타내는 행이라 회차 등록 시점에 미리 생성된 것([[../architecture/db-schema-conventions]] 참고), 지우면 그 슬롯 자체가 없어짐.

### 3. 확인

```sql
SELECT reserved_status, COUNT(*) FROM RESERVATIONS WHERE round_id = <ROUND_ID> GROUP BY reserved_status;
-- 전부 AVAILABLE이어야 함
```

## 주의사항

- **prod는 실제 고객이 쓸 수 있는 환경** — round_id를 고를 때 이미 실서비스에 열려있는 회차가 아니라 테스트 전용 회차를 쓰는 게 이상적이나, 지금은 팀 내부 테스트용으로 기존 회차(round_id 11 등)를 재사용하고 있어서 매번 이 초기화가 필요함.
- `queue:*` 전체 삭제는 **그 Redis 인스턴스 전체의 모든 회차 대기열**을 지운다 — 동시에 다른 round_id로 진행 중인 실제 대기열이 있다면 그것도 같이 지워짐. 필요하면 `queue:{roundId}:*`처럼 특정 round만 패턴을 좁힐 것.
- k6 스크립트 자체의 `reservation_unexpected_fail` threshold(`count<20`)나 `ROUND_ID` 기본값은 [[../troubleshooting/backend-cold-start-cpu-contention-during-rollout]] 참고 — 이 절차와 별개로 스크립트 쪽 설정도 매번 확인.

## 관련
- [[../troubleshooting/backend-cold-start-cpu-contention-during-rollout]]
- [[../troubleshooting/loadtest-10000-open-run-cascading-failures]]
- [[../architecture/db-schema-conventions]]
