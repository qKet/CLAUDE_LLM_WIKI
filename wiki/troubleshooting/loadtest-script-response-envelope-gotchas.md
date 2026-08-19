---
title: k6 부하테스트 스크립트가 GlobalResponseAdvice 응답 래핑 규칙을 몰라서 계속 undefined였던 문제
category: troubleshooting
tags: [load-test, k6, api-response-format]
created: 2026-08-19
updated: 2026-08-19
---

# k6 부하테스트 스크립트가 GlobalResponseAdvice 응답 래핑 규칙을 몰라서 계속 undefined였던 문제

## 증상

`e2e_reservation_2000.js`(로그인→대기열→좌석선택→예매 시나리오)를 여러 번 돌렸는데 매번 예매 성공률이 0%에 가까웠음. [[queue-max-active-users-bottleneck]]을 고친 뒤에도 여전히 안 됨 — `/schedules/.../seats`, `/reservations` 요청 자체는 나가는데 다 실패.

## 원인

두 가지가 겹쳐 있었음.

**1) 응답 래핑 규칙을 놓침** — `GlobalResponseAdvice`([[../conventions/api-response-format]])는 컨트롤러가 **Map을 직접 리턴하면 그대로 두고**(`success`/`message`가 최상위), **DTO/record/List를 리턴하면 `{success, message, data, timestamp}`로 감싸서** `data` 밑에 넣음. `/queues`(`QueueJoinResponse` record), `/queues/{token}`(`QueueStatusResponse` record), `/schedules/.../seats`(`List<SeatDTO>`)는 전부 후자라 `data.queueToken`, `data.status`, `data` 자체가 배열인데, 스크립트는 최상위에서 바로 `.queueToken`/`.status`를 읽고 있어서 계속 `undefined`였음. `/reservations`는 `Map`을 직접 리턴해서 최상위 `success`가 맞았던 것 — 그래서 이 필드 하나만 "우연히" 맞고 나머지는 다 틀렸었음.

**2) 좌석 목록 API의 `roundId`가 항상 `null`** — `/schedules/{roundId}/seats` 응답의 `SeatDTO.roundId`가 매퍼 쿼리에서 채워지지 않아 항상 `null`로 내려옴. 1번을 고친 뒤에도, 이 `null`을 그대로 예매 요청(`POST /reservations`)에 넣으면 백엔드의 `UPDATE ... WHERE round_id=?` 조건이 `NULL`과 비교돼서 절대 매칭이 안 됨(SQL에서 `NULL = NULL`은 `NULL`이지 `TRUE`가 아님) — 매번 "이미 예매된 좌석"처럼 실패.

## 해결

`e2e_reservation_2000.js`:
```diff
- const queueToken = JSON.parse(joinRes.body).queueToken;
+ const queueToken = JSON.parse(joinRes.body).data.queueToken;

- const status = JSON.parse(statusRes.body).status;
+ const status = JSON.parse(statusRes.body).data.status;

- const seats = JSON.parse(seatsRes.body);
+ const seats = JSON.parse(seatsRes.body).data;

- roundId: pick.roundId,
+ roundId: ROUND_ID, // 응답의 roundId가 항상 null이라 알고 있는 상수로 우회
```

## 재발 방지

- **API 클라이언트(k6 스크립트든 프론트든)를 새로 짤 때, 해당 엔드포인트의 실제 응답을 `curl`로 한 번 찍어보고 시작할 것** — 컨트롤러 코드만 보고 리턴 타입을 추측하면 이번처럼 래핑 여부를 놓치기 쉬움. 이번에도 스크립트를 만들 때 컨트롤러/DTO 코드만 보고 짰다가, 실제 curl로 찍어보고서야 구조를 알았음.
- `SeatDTO.roundId`가 항상 null인 것 자체는 이번엔 프론트가 애초에 그 값을 안 쓰기 때문에(요청 시 알고 있는 라운드ID를 씀) 실제 버그로 이어지진 않았지만, API 응답에 있는 필드인데 항상 null인 건 그 자체로 매퍼 쿼리 점검 대상.

## 관련
- [[../conventions/api-response-format]]
- [[queue-max-active-users-bottleneck]]
