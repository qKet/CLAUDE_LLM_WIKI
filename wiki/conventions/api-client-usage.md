---
title: API 호출 규칙 (apiFetch)
category: conventions
tags: [frontend, api]
created: 2026-08-06
updated: 2026-08-06
---

# API 호출 규칙

## 규칙

모든 API는 `lib/api/client.ts`의 `apiFetch<T>(path, options)`를 통해 호출한다.

- 자동으로 `credentials:"include"`(세션 쿠키), JSON 직렬화/역직렬화.
- 백엔드가 `{success,data,timestamp}` 모양으로 감싸서 응답하면 `data`만 자동으로 꺼내줌(`unwrap()`) — 호출부는 신경 안 써도 됨.
- 실패 시 `ApiError`(code/status/errors 포함)를 throw.
- `apiFetch`를 안 거치고 서버 컴포넌트에서 직접 `fetch()`를 쓰는 곳(절대경로 URL 호출)이 있다면, 거기서도 `unwrap()`을 직접 불러다 써야 함 — **안 그러면 배열인 줄 알고 `.map()`했다가 터지는 버그가 남**.

## 왜 이렇게 하나

백엔드 응답 규격([[api-response-format]])이 `{success,data,timestamp}`로 감싸져 오기 때문에, 모든 호출부가 매번 언래핑 로직을 반복하지 않도록 `apiFetch` 한 곳에 모아뒀다. 이 관문을 우회하면 언래핑이 누락되어 런타임 에러로 이어진다.

## 관련
- [[api-response-format]]
- [[frontend-folder-structure]]
