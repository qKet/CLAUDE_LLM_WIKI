---
title: 관리자 그리드 페이지 패턴
category: conventions
tags: [frontend, admin]
created: 2026-08-06
updated: 2026-08-06
---

# 관리자 그리드 페이지 패턴

대상: `app/admin/*/page.tsx` (예: `users`, `programs`, `menus`)

## 규칙

- `changes: Record<id, Partial<Row>>` state로 dirty-tracking, "저장" 버튼으로 한 번에 배치 저장.
- 필드가 null이 될 수 있으면(예: `parentMenuId`) `getVal`을 `?? original`(nullish coalescing)이 아니라 **`field in changes[id]`로 판단**해야 함 — `??`는 "명시적으로 null로 바꿈"과 "안 바꿈(undefined)"을 구분 못 해서 화면에 반영이 안 되는 버그가 생긴다.
- 저장 시에도 마찬가지로, null 허용 필드가 있는 행은 변경된 필드만 보내지 말고 **그 행의 최종 상태 전체**를 합쳐서 보내야 한다 (안 그러면 [[mybatis-conventions]]에서 설명한 이유로 서버가 안 보낸 필드를 null로 덮어씀).

## 왜 이렇게 하나

이 두 규칙은 백엔드의 MyBatis nullable FK 처리 방식과 짝을 이룬다. 프론트가 "부분 diff"만 보내는 걸 전제로 백엔드를 짜면 null 허용 필드에서 값이 사라지므로, 프론트도 항상 "이 행의 완전한 최종 상태"를 보내는 쪽으로 계약을 맞췄다. 실제로 겪은 버그 사례는 [[null-field-partial-update-bug]] 참고.

## 관련
- [[mybatis-conventions]]
- [[null-field-partial-update-bug]]
