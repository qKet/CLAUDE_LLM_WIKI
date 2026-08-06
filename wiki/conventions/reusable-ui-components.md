---
title: 재사용 UI 컴포넌트
category: conventions
tags: [frontend, ui]
created: 2026-08-06
updated: 2026-08-06
---

# 재사용 UI 컴포넌트

위치: `frontend/components/ui/`

반복되는 UI 패턴은 raw `className`을 새로 조합하지 말고 아래 컴포넌트를 우선 재사용한다. 전부 기존 CSS 클래스/값을 그대로 감싸기만 한 것이라 스타일 자체는 안 바뀐다.

## 컴포넌트 목록

- **`Button`** — `variant`(`primary`/`secondary`/`ghost`/`danger`) + `fullWidth`.
  - `primary` = 화면당 핵심 행동 1개(예매하기/제출)
  - `secondary` = 보조 행동(수정/닫기)
  - `ghost` = 배경 없이 테두리만
  - `danger` = 되돌리기 어려운 행동(삭제/취소)
- **`Badge`** — `variant`(`open`/`closed`/`soldout`/`vip`/`r`/`s`).
- **`FormField` + `Input`** — `variant`(`auth`/`admin`)로 각각 로그인·회원가입 폼(`field`/`fieldInput`)과 관리자 폼(`adminFormRow`/`adminInput`) 스타일을 고름.
  - `FormField`는 라벨+래퍼만 담당하고 실제 입력 요소는 `children`으로 받음(`select` 등 `Input`이 아닌 요소도 가능).
  - `Input`은 관리자 폼 유효성 검사 실패 시 focus 이동을 위해 `forwardRef` 지원.
- **`PageHeader`** — 페이지 최상위 wrapper(`pageWrap`/`pageWrapWide`)까지 함께 감싸므로 페이지 내용 전체를 `children`으로 넘긴다. `title`/`subtitle`/`variant`(`default`/`admin`)/`wide`/`actions`(우측 버튼 영역) prop 사용. 로딩 중 조기 return처럼 제목 없이 문구만 있는 자리에는 안 맞으니 그대로 `pageWrap` div를 쓴다.
- **`StatusMessage`** — `variant`(`loading`/`error`/`success`), 기본 태그 `<p>`, 인라인 배치가 필요하면 `as="span"`.

## 관련
- [[css-file-structure]]
