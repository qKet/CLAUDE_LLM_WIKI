---
title: 세션-대기열 Redis 인스턴스 공유 리스크 및 대응 방향
category: decisions
status: 논의중
date: 2026-08-10
author: MoonJunH
tags: [redis, elasticache, session, queue, capacity-planning]
updated: 2026-08-18
---

# 세션-대기열 Redis 인스턴스 공유 리스크 및 대응 방향

> 2026-08-18: [[2026-08-18-capacity-planning-large-traffic-readiness]](대용량 트래픽 용량 분석)에서 Redis 스펙 상향도 후보로 나왔으나, 그 결정을 이 문서로 다시 미룸 — 지금 스펙만 올려봐야 "재검토 트리거"(부하테스트 실측)가 아직 없어서 근거 없이 비용만 느는 것이고, 대기열 기능을 실제로 구현하는 시점에 대안 A(물리 분리)로 갈 가능성이 있어 그때 같이 정하는 게 낫다는 판단(사용자 확인). RDS는 같은 분석에서 release만 `db.t3.medium`으로 먼저 올림 — Redis만 별도로 보류.

## 배경

현재 세션(Spring Session)과 대기열(`QueueService`)이 같은 ElastiCache Redis 인스턴스를 공유하고 있는데, 대기열에 부하가 몰렸을 때(티켓 오픈런 등) 이 구조가 문제없을지 검토 요청이 있었음.

### 확인된 사실

**1. Redis 자체가 이미 최소 스펙, 싱글 노드**
- `terraform/release/variables.tf`: `redis_node_type = "cache.t3.micro"` (release는 싱글 노드, 최소 사양)
- `terraform/modules/data/elasticache.tf`: `num_cache_nodes = 1` (자동 페일오버 없음, prod도 아직 안 켜짐 — `terraform/prod/data.tf`에 통째로 주석 처리돼 있음)
- `t3.micro`는 버스트 크레딧 기반의 가장 작은 인스턴스, 복제본/클러스터 모드 없이 단일 노드

**2. 세션과 대기열이 물리적으로 같은 Redis 인스턴스를 공유**
- 세션: `RedisConfig`의 `LettuceConnectionFactory`(host/port 고정) → `@EnableRedisHttpSession`으로 Spring Session이 이걸로 Redis에 씀
- 대기열: `RedisQueueRepository`가 `StringRedisTemplate` 사용 → 얘도 결국 같은 `RedisConnectionFactory`, 같은 엔드포인트
- 네임스페이스(`queue:*` vs 세션 키)만 다를 뿐 같은 노드, 같은 CPU, 같은 메모리를 씀

**3. 대기열 트래픽 패턴 자체가 폭발적**
- 프론트 `QueueModal.tsx`: `setInterval(() => poll(queueToken), 3000)` — 대기자 전원이 3초마다 폴링
- `QueueServiceImpl.getStatus()` 호출마다 매번 `admitAvailableUsers()`를 같이 호출하는 구조라, 요청 하나당 Redis 커맨드가:
  - 세션 조회 (요청마다 Spring Session이 Redis 터치)
  - `findToken` (HGET × 2)
  - `acquireAdmissionLock` (SETNX)
  - 락 획득 시: `removeExpiredActive`(ZREMRANGEBYSCORE), `getActiveCount`(ZCARD), `getFirstWaiting`(ZRANGE), `moveToActive`(ZREM+ZADD), `refreshToken`/`refreshUserToken`(EXPIRE) 다발
  - `getWaitingRank` (ZRANK)
  - → 요청당 최소 4~5개 Redis 커맨드
- 예시 계산: 티켓 오픈 순간 대기자 5,000명 → 초당 약 1,600건 요청 × 커맨드 4~5개 = **초당 8,000개 안팎의 Redis 명령어**가 대기열 하나에서만 발생. `t3.micro`의 버스트 크레딧으로는 이 정도 지속 부하를 감당하기 빠듯함
- 참고: `queue:{scheduleId}:...`처럼 키에 해시태그를 넣어둔 걸 보면 나중에 클러스터 모드를 염두에 둔 설계는 되어 있음 (지금은 싱글 노드라 아직 실질적 의미는 없음)

## 결정

**오토 페일오버는 운영(prod) 전환 시점에 켜기로 확정.** 지금은 예산 문제로 보류 — `dev`/`release`는 싱글 노드 유지, prod 전환 시 `aws_elasticache_replication_group` + `num_cache_clusters >= 2`로 전환 예정 (`terraform/modules/data/elasticache.tf` 주석에 이미 명시돼 있던 계획).

**세션/대기열 Redis 물리 분리 여부는 아직 미확정.** 아래 트레이드오프 표에서 보듯 페일오버만으로는 해결 안 되는 리스크(noisy neighbor, 메모리 압박)가 남아있지만, 분리는 순수 추가 비용(인스턴스 증설)이라 "얼마나 위험한지"를 실측 없이 바로 결정하지 않기로 함. **실제 오픈런 트래픽을 흉내낸 부하테스트로 `t3.micro` 싱글 노드의 실제 한계치를 먼저 확인하고, 그 결과를 보고 물리 분리 여부를 재검토**하기로 함.

## 고려한 대안

- **A. 세션/대기열 Redis 물리 분리** — `terraform/modules/data` 모듈을 한 번 더 호출해서 대기열 전용 ElastiCache 추가, 백엔드는 `RedisConfig` 옆에 큐 전용 `ConnectionFactory`를 추가해서 `RedisQueueRepository`가 그걸 쓰도록 변경. Noisy neighbor·메모리 압박·정책 차등(세션은 `noeviction`, 큐는 상대적으로 덜 방어적으로) 문제를 모두 해결하지만, 인스턴스가 늘어나는 만큼 무조건 비용 증가.
- **B. 논리 DB 분리** (`SELECT` 0~15, 클러스터 모드 아닐 때만 가능) — 세션용 `ConnectionFactory`는 `setDatabase(0)`, 큐용은 `setDatabase(1)`처럼 인스턴스는 그대로 두고 논리적으로만 나눔. **비용 0원**이지만, `maxmemory-policy`와 CPU는 노드 단위 리소스라 논리 DB를 나눠도 여전히 물리적으로 같은 노드를 같이 씀 — 키 네임스페이스 정리 효과만 있고 실제 리소스 격리는 안 됨. 지금 채택하지 않았지만 언제든 무비용으로 적용 가능한 최소한의 조치로 남겨둠.
- **C. 페일오버만 켜고 분리는 보류** (현재 결정) — 장애 전파 리스크만 우선 해소하고, noisy neighbor·메모리 압박 리스크는 부하테스트로 실측 후 재검토.

## 트레이드오프 / 남은 리스크

### 페일오버로 해결되는 것 vs 안 되는 것

| 리스크 | 페일오버로 해결? |
|---|---|
| 노드 장애 시 큐+세션 동시 마비 | ✅ 해결됨 (레플리카 자동 승격) |
| 대기열 폭주 → 세션 응답 지연 (noisy neighbor) | ❌ 그대로 남음 — 읽기/쓰기는 기본적으로 프라이머리 한 대가 처리, 레플리카는 장애 대비용이지 부하 분산용이 아님 |
| 메모리 압박 → 세션 eviction | ❌ 그대로 남음 — 레플리카 추가는 노드 타입을 안 키우면 메모리 용량 자체엔 도움 안 됨 (오히려 복제 부담으로 약간 더 빡빡해질 수 있음) |

### 실제 리스크 상세

- **Noisy neighbor** — Redis는 사실상 커맨드 처리가 단일 스레드라, 대기열 폭주로 노드가 바빠지면 결제하려는 로그인 유저의 세션 조회도 같이 느려짐/타임아웃. 하필 트래픽이 제일 몰릴 때(오픈런) 사이트 전체 인증이 느려지는 최악의 타이밍.
- **장애 전파 범위** — `num_cache_nodes = 1`(페일오버 켜기 전까지)이라 노드가 죽거나 재시작되면 대기열뿐 아니라 로그인 세션도 동시에 날아감.
- **메모리 압박 → 세션 eviction 가능성** — 대기자 많아지면 큐 관련 키가 급증하는데, `t3.micro` 메모리 한도에 걸리면(eviction 정책에 따라) 결제 중이던 유저 세션이 강제로 밀려나 로그아웃되거나, 쓰기 자체가 실패해서 큐/세션 둘 다 500 에러가 날 수 있음.

### 비용 구조 비교 (분리는 "절감"이 아니라 "추가 비용"임에 유의)

| 구성 | 노드 수 |
|---|---|
| 지금 (분리 X, 페일오버 X) | t3.micro × 1 |
| 페일오버만 켜기 (확정된 계획) | t3.micro × 2 (프라이머리+레플리카) |
| 분리 + 페일오버까지 | t3.micro × 4 (세션 2대 + 큐 2대) |

분리하면서 페일오버까지 다 하면 지금 대비 노드가 4배. 정확한 금액은 AWS Pricing Calculator로 확인 필요 — 방향성만 "분리 = 추가 비용, 절감 아님"으로 명확히 해둠 (분리하면 더 싸질 거라는 오해가 있었어서 정정한 기록).

### 재검토 트리거

- 부하테스트로 `t3.micro` 싱글 노드가 실제 대기자 규모(폴링 3초 간격 기준)에서 어느 지점부터 지연/에러가 나는지 확인 → 임계치가 실제 예상 트래픽보다 낮으면 물리 분리(대안 A)로 전환
- 운영 전환(prod) 시점에 페일오버를 켜는 작업과 묶어서, 그때 인프라를 손대는 김에 분리도 같이 반영하는 것도 검토 가능

## 관련
- [[../architecture/auth-and-authorization]] — 세션이 Redis-backed `HttpSession`이라는 현재 구조
- [[../architecture/terraform-module-boundaries]] — `modules/data`(RDS/ElastiCache) 모듈 경계
- [[../architecture/terraform-platform-workload-split]] — platform/workload 분리 구조, prod 전환 시 이 결정과 함께 반영될 여지
