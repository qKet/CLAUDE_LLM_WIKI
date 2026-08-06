---
title: CSS 파일 구조
category: conventions
tags: [frontend, css]
created: 2026-08-06
updated: 2026-08-06
---

# CSS 파일 구조

작업 로그: `docs/css-refactor-log.md`

## 구조

`app/globals.css`는 조립 창구일 뿐이고, 실제 스타일은 전부 `frontend/styles/*.css`에 역할별로 분리돼 있다 (`@import`만 나열).

```
styles/
 ├─ base.css       (reset, :root 변수)
 ├─ nav.css        (상단 네비게이션)
 ├─ layout.css     (페이지 공통 레이아웃 틀)
 ├─ auth.css       (로그인/회원가입 + 폼)
 ├─ button.css     (버튼 4종)
 ├─ card.css       (카드 / 공연 목록 그리드)
 ├─ badge.css      (상태 뱃지)
 ├─ queue.css      (대기열 화면)
 ├─ seat.css       (좌석 선택 화면)
 ├─ mypage.css     (마이페이지)
 ├─ message.css    (공통 에러/성공/로딩 메시지)
 ├─ admin.css      (관리자 전체 화면)
 └─ responsive.css (반응형 미디어쿼리, 항상 마지막 import)
```

## 규칙

- 새 화면/기능의 스타일은 성격이 맞는 기존 파일에 추가한다.
- 어디에도 안 맞으면 새 역할 파일을 만들고 `globals.css`에 `@import` 한 줄 추가 — 단 `responsive.css` import보다 **앞에** 둔다 (반응형이 항상 마지막에 적용되어야 함).
- 기존 클래스명(`btnPrimary`, `pageWrap`, `adminModalOverlay` 등)과 값은 이 분리 작업으로 전혀 안 바뀌었음 — 파일 위치만 재배치된 것.

## 관련
- [[reusable-ui-components]]
