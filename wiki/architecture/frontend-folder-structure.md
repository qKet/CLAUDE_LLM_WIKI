---
title: 프론트엔드 폴더 구조
category: architecture
tags: [frontend]
created: 2026-08-06
updated: 2026-08-06
---

# 프론트엔드 폴더 구조

백엔드 패키지 경계를 그대로 미러링한다.

- `lib/api/<domain>/index.ts` — 도메인별 API 함수 (예: `lib/api/admin/`, `lib/api/manage/`(공연 관리, `/manage/*`), `lib/api/common/`(업로드, 메뉴 조회))
- `lib/data/types/<domain>.ts` — 도메인별 TypeScript 타입, `lib/data/types/index.ts`에서 전부 barrel export. **새 도메인 타입 추가 시 여기 등록.**

## 왜 이렇게 하나

백엔드 `com.exam.<domain>` 패키지와 프론트 `lib/api/<domain>`이 1:1로 대응되도록 맞춰서, 어느 한쪽을 보고 있어도 반대쪽 위치를 바로 유추할 수 있게 한다.

## 관련
- [[backend-layer-and-package-structure]]
- [[api-client-usage]]
