---
title: 로컬 DB 스키마 변경 절차
category: runbook
tags: [database, docker]
created: 2026-08-06
updated: 2026-08-06
---

# 로컬 DB 스키마 변경 절차

## 배경

`spring.sql.init.mode: never`로 설정되어 있어, Spring Boot는 `schema.sql`/`data.sql`을 직접 실행하지 않는다. 이 파일들은 `docker-entrypoint-initdb.d`를 통해 **완전히 새 볼륨일 때만** 자동 실행된다.

## 절차

로컬 개발 중 스키마를 바꿀 땐 두 가지 방법 중 하나를 쓴다.

### 방법 1: 완전 재생성

```bash
docker compose down -v && docker compose up -d
```

기존 데이터가 다 날아가고 시드(`data.sql`)로 복구된다. 스키마 변경이 크거나 데이터가 아까울 게 없을 때.

### 방법 2: 떠 있는 컨테이너에 직접 반영

```bash
docker exec -i qket-mysql mysql --default-character-set=utf8mb4 -uroot -p1234 qket -e "..."
```

**`--default-character-set=utf8mb4`를 반드시 붙인다** — 안 붙이면 한글이 깨진다. 자세한 사고 경위는 [[utf8mb4-encoding-bug]] 참고.

## 주의사항

**어느 방법을 쓰든, 바꾼 내용은 `schema.sql`/`data.sql`에도 반드시 반영해서 파일과 라이브 DB가 어긋나지 않게 유지한다.** `docker exec`로 DB에 직접 낸 변경사항을 파일에 옮기는 걸 잊기 쉬우니 주의 — 다음에 `down -v`로 재생성하면 그 변경이 통째로 사라진다.

## 관련
- [[db-schema-conventions]]
- [[utf8mb4-encoding-bug]]
