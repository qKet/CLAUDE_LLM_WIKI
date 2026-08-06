---
title: DB 스키마 컨벤션
category: architecture
tags: [backend, database]
created: 2026-08-06
updated: 2026-08-06
---

# DB 스키마 컨벤션

파일: `backend/src/main/resources/schema.sql`, `data.sql`

## 규칙

모든 테이블에 감사 컬럼 6개를 포함한다:

- `ins_id` (기본값 `'SYSTEM'`, 로그인 전 행위엔 이 값)
- `ins_ip`
- `ins_de`
- `upt_id`
- `upt_ip`
- `upt_de` (`ON UPDATE CURRENT_TIMESTAMP`)

**새 테이블 추가 시 이 6개 컬럼을 그대로 포함시킨다.**

`ins_ip`/`upt_ip`가 채워지는 방식(매 요청마다 라이브로 IP 추출)은 [[auth-and-authorization]] 참고.

## 관련
- [[auth-and-authorization]]
- [[db-schema-change]] (runbook)
