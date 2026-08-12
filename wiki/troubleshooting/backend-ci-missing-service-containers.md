---
title: backend CI에 MySQL/Redis 서비스 컨테이너가 없어서 테스트 실패
category: troubleshooting
tags: [ci, github-actions, backend, mysql, redis, test]
created: 2026-08-11
updated: 2026-08-11
---

# backend CI에 MySQL/Redis 서비스 컨테이너가 없어서 테스트 실패

## 증상

`backend/.github/workflows/CI-release.yml`의 `build-and-push` 잡에서 `DemoApplicationTests.contextLoads()`가 `RedisConnectionException`으로 실패. MySQL 연결도 실패했을 것으로 보이나 Redis가 먼저 걸림.

## 원인

GitHub-hosted runner는 기본적으로 빈 컨테이너라 MySQL/Redis가 없는데, 백엔드 스프링 컨텍스트가 뜨는 과정(`contextLoads`)에서 `application.yml`의 DB/Redis 설정으로 실제 연결을 시도하기 때문에 둘 다 없으면 바로 실패한다.

## 해결

CI 워크플로우에 GitHub Actions `services:` 블록으로 MySQL/Redis 컨테이너를 추가:

```yaml
services:
  mysql:
    image: mysql:8.0
    env:
      MYSQL_ROOT_PASSWORD: "1234"
      MYSQL_DATABASE: qket
    ports: ["3306:3306"]
    options: >-
      --health-cmd="mysqladmin ping" --health-interval=10s
      --health-timeout=5s --health-retries=5
  redis:
    image: redis:7
    ports: ["6379:6379"]
    options: >-
      --health-cmd="redis-cli ping" --health-interval=10s
      --health-timeout=5s --health-retries=5
```

값(`root`/`1234`/`qket`, `localhost`)은 `application.yml`의 로컬 기본값과 그대로 일치시켜서 **애플리케이션 코드는 전혀 안 건드림** — CI 러너 안에서 `localhost:3306`/`localhost:6379`로 붙는 빈 컨테이너일 뿐이다.

## 왜 실제 RDS/ElastiCache에 붙이지 않았나

처음엔 "CI가 실제 인프라(RDS/Redis)에 연결하면 되지 않나"는 방향도 검토했으나 기각:

- **네트워크 격리**: RDS/ElastiCache는 프라이빗 서브넷에 있고 GitHub-hosted runner에서 도달 불가 — 도달시키려면 VPN/피어링 같은 별도 인프라가 필요해짐
- **데이터 오염 리스크**: 테스트가 실제 운영/스테이징 데이터베이스에 쓰기 작업을 하게 됨
- **결합도**: CI 성공 여부가 [[../runbook/daily-infrastructure-toggle]]로 매일 밤 껐다 켜는 인프라 상태에 종속됨 — 밤엔 인프라가 없으므로 그 시간대 CI가 항상 실패하게 됨

빈 서비스 컨테이너로 "연결 자체가 되는지"만 검증하는 지금 방식이 CI의 목적(빌드/컨텍스트 로딩 검증)에 맞다고 판단.

## 재발 방지

- 새로운 외부 연결(다른 DB, 메시지 큐 등)이 `contextLoads`/통합 테스트 경로에 들어가면, 항상 GitHub Actions `services:`로 같은 값을 로컬 기본값에 맞춰 빈 컨테이너를 띄우는 패턴을 우선 검토한다.

## 관련
- [[cd-helm-chart-deploy-review]]
- [[../runbook/daily-infrastructure-toggle]]
