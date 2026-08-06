---
title: 동적 메뉴/권한 시스템
category: architecture
tags: [frontend, backend]
created: 2026-08-06
updated: 2026-08-06
---

# 동적 메뉴/권한 시스템

## 구조

`PROGRAMS`/`ROLE_PROGRAMS`/`MENUS` 테이블 기반으로 `SiteNav.tsx`가 로그인 사용자 role에 맞는 메뉴 트리를 `GET /common/menus/my`로 받아와 렌더링한다 (하드코딩된 role 체크가 아님).

`MENUS.program_id`가 `NULL`이면 "그룹 전용" 메뉴다 — 자기 자신은 클릭할 페이지 없이 하위메뉴만 호버로 노출 (예: 상단 "관리자" 드롭다운).

## 적용 범위 (중요)

**이건 화면(메뉴 노출/페이지 접근)에만 적용되는 시스템이고, 백엔드 API 자체의 권한 체크는 별개다.** 각 컨트롤러가 하드코딩된 role 체크로 API를 직접 지킨다 — 자세한 내용은 [[auth-and-authorization]] 참고.

이 둘을 혼동하면 "메뉴에 안 보이니까 안전하다"고 착각하기 쉬운데, 메뉴 숨김은 UX일 뿐 보안 경계가 아니다.

## 관련
- [[auth-and-authorization]]
- [[backend-layer-and-package-structure]]
