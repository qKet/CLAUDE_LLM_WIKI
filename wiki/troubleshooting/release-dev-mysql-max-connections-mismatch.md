---
title: release backend maxReplicas(20)가 dev-mysql의 max_connections(151)를 초과해 있었음
category: troubleshooting
tags: [release, dev-datastore, mysql, hikaricp, keda, autoscaling, cd, values]
created: 2026-08-21
updated: 2026-08-21
---

# release backend maxReplicas(20)가 dev-mysql의 max_connections(151)를 초과해 있었음

## 증상

실제 장애로 터진 건 아니고, `CD/helm/values-prod.yaml`을 release와 비교해서 정리하던 중 사용자가
"이거 240개가 생겨버리는데 괜찮은거야?"라고 직접 지적하면서 발견됨.

`CD/helm/values-release.yaml`의 `backend.dbPoolSize(12) × backend.autoscaling.maxReplicas(20)
= 240` — 이 240이라는 상한이 실제 DB의 커넥션 한도보다 큰 상태로 방치돼 있었음.

## 원인

`maxReplicas: 20`(정확히는 `dbPoolSize 12`와 짝지어 `240`)은 원래 2026-08-13 부하테스트 사고
([[hikaricp-connection-storm-load-test]] 참고)를 계기로, release가 RDS(`db.t3.medium`,
`max_connections≈341`, RDS 파라미터그룹 공식값)를 쓰던 시절에 "341의 약 70%"로 역산해서 잡은
값이었음.

그런데 [[2026-08-21-release-datastore-rds-to-statefulset]]로 release가 RDS를 완전히 버리고
`dev-datastore`(StatefulSet, `mysql:8.0` 이미지를 커스텀 설정 없이 그대로 씀)로 옮겨간 뒤, 이
`240`이라는 숫자의 근거(db.t3.medium의 341)가 통째로 사라졌는데 값 자체는 재계산 없이 그대로
남아있었음. `dev-mysql-0` pod에 직접 접속해 확인한 실제 값:

```sql
SHOW VARIABLES LIKE 'max_connections';   -- 151 (MySQL 8.0 기본값, my.cnf 커스텀 없음)
SHOW STATUS LIKE 'Max_used_connections'; -- 62 (실측 피크)
```

즉 `240 > 151`로 이미 초과 상태였음. 지금까지 실제 장애로 안 터진 이유는 KEDA가 CPU 기준으로
아직 20 replica까지 실제로 스케일업한 적이 없었기 때문(`Max_used_connections` 실측 피크가 62에
불과 — replica 4~5개 수준의 부하까지만 실제로 겪어봤다는 뜻)일 뿐, 언젠가 CPU 부하가 늘어
KEDA가 진짜로 20까지 스케일업했다면 2026-08-13과 같은 클래스의 HikariCP 커넥션 고갈 장애가
release에서도 재현됐을 것.

## 해결

1차로는 `backend.autoscaling.maxReplicas`를 `20 → 8`로 낮춰서(`dbPoolSize` 12 유지, `12×8=96`,
151의 약 64%) 커넥션 초과 문제만 우선 해결했음.

같은 날 후속으로 사용자가 "release는 어차피 개발 서버니까 최대한 가볍게, 꼭 필요한 만큼만
띄우고 싶다"고 방향을 더 좁혀서, 안전 마진(8)보다 더 낮춰 **`minReplicas: 1 / maxReplicas: 2`**로
최종 조정함(backend/frontend 둘 다, `replicas` 초기값도 4→1로 같이 낮춤). `12×2=24`로 151 대비
여유는 훨씬 커졌지만, 대신 이 낮은 replica 수 자체가 release의 실질 처리량 상한이 됨 — 팀원이
직접 부하테스트를 하려면 이 값을 일시적으로 올려야 한다는 트레이드오프가 생김. `helm template`로
렌더링 확인, 에러 없음.

prod는 이번에 안 건드림(`db.t3.small`, `max_connections≈170` 기준 `dbPoolSize 12 × maxReplicas 8
= 96`을 그대로 유지) — prod는 "개발 서버처럼 가볍게"라는 이번 방침 대상이 아님.

## 재발 방지

- release의 dev-mysql/dev-redis 같은 "RDS가 아닌 자체 호스팅 DB"로 환경을 바꿀 때는, DB
  인스턴스 클래스가 바뀌는 것과 동일하게 `dbPoolSize × maxReplicas` 재계산이 필요하다는 걸
  체크리스트로 남겨야 함(RDS 인스턴스 클래스 변경 시에는 이미 이 계산을 다시 하는 습관이
  있었는데, "RDS 자체를 다른 종류의 DB로 교체"하는 경우는 놓치기 쉬웠음).
- `dev-mysql`은 `my.cnf` 커스텀이 전혀 없어서 `max_connections`가 MySQL 기본값(151)에 그대로
  묶여있음 — 나중에 release 트래픽이 늘어서 8 replica로도 부족해지면, `maxReplicas`를 더
  올리기 전에 `modules/addons/dev-datastore`의 MySQL 컨테이너에 `--max-connections` 같은
  커스텀 설정을 먼저 추가해야 함(단, 커넥션당 메모리 사용이 늘어나므로 현재 1Gi 메모리
  limit도 같이 재검토 필요).

## 관련
- [[../decisions/2026-08-21-release-datastore-rds-to-statefulset]]
- [[hikaricp-connection-storm-load-test]]
- [[backend-cpu-throttling-and-scaling-load-test]]
- [[../decisions/2026-08-18-capacity-planning-large-traffic-readiness]]
