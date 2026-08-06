---
title: 주석 규칙 (백엔드/프론트엔드)
category: conventions
tags: [backend, frontend, comment]
created: 2026-08-06
updated: 2026-08-06
---

# 주석 규칙

## 규칙

### 백엔드: Controller/Service/ServiceImpl 메서드 주석

각 public 메서드 바로 위에 아래 형식의 블록 주석을 단다 (실제 예: `AdminController.java`의 `getRoles`/`updateUser`/`batchUpdateUsers`).

```
/***********************************
 * URL : "/users/batch"
 * 이름 : 사용자 정보 일괄 수정
 * 기능 : 관리자가 사용자 상태 및 권한 일괄 수정
 * method : Patch
 ************************************/
```

- `URL`/`method`는 그 로직이 속한 엔드포인트 기준(예: `@PatchMapping("/users/batch")`) — Service/ServiceImpl 메서드에도 그 로직을 실제로 호출하는 Controller 엔드포인트와 **동일한 URL/method**를 적어서, 레이어를 넘나들며 어떤 API의 구현인지 바로 추적할 수 있게 한다.
- `이름`은 한 줄 기능명, `기능`은 누가/왜 하는지까지 포함한 조금 더 구체적인 설명.
- 아직 `AdminController`/`ProgramController`/`MenuController`/`CommonController`에만 적용돼 있고 나머지 컨트롤러·모든 Service/ServiceImpl에는 없음 — 전체 소급 적용 대상이 아니라, **새로 추가하거나 수정하는 메서드부터** 이 규칙을 따른다.

### 프론트엔드: API 함수 주석

`lib/api/<domain>/*.ts`(barrel `index.ts` 제외)의 각 API 함수 바로 위에 아래 형식의 블록 주석을 단다 (실제 예: `lib/api/auth.ts`의 `login`, `lib/api/events.ts`의 `getEvents`).

```
// ============================================================
// GET /api/events
// 백엔드: PerformanceController.java → list()
// 기능: 전체 공연 목록 조회 (메인/목록 화면용)
//
// 사용 예시:
//   import { getEvents } from "@/lib/api/events";
//
//   useEffect(() => {
//     getEvents().then(setPerformances).finally(() => setLoading(false));
//   }, []);
//
// 요청: 파라미터 없음
// 응답 JSON (Performance[] — 공연마다 rounds 배열까지 포함해서 옴):
//   [
//     {
//       "performanceId": 1,
//       "pTitle": "뮤지컬 지킬앤하이드",
//       "pLocation": "고척스카이돔",
//       "posterUrl": "https://.../poster.jpg",
//       "rounds": [
//         { "roundId": 10, "performanceId": 1, "roundTime": "2026-08-15 19:00:00",
//           "openTime": "2026-08-01 10:00:00", "roundStatus": "OPEN" }
//       ]
//     }
//   ]
// ============================================================
```

- 첫 줄은 `METHOD /api/경로`, 그다음 `백엔드:` 줄에 실제로 이 API를 처리하는 `컨트롤러파일.java → 메서드명()`을 적어서 프론트-백엔드 코드를 바로 대응시킬 수 있게 한다.
- `사용 예시`는 호출부에서 실제로 쓰는 모양(어떤 훅/컴포넌트에서, 어떻게 결과를 받는지) 그대로 붙여넣는다 — 추상적인 설명이 아니라 복붙 가능한 코드.
- body가 있는 요청(POST/PATCH 등)은 `요청 JSON (프론트 → 백엔드, body)`, 없으면 `요청: 파라미터 없음` 또는 path/query 파라미터를 명시.
- `응답 JSON`은 실제 필드명과 타입을 알 수 있는 예시 값으로 적는다 (타입 정의만 보고는 실제 모양이 바로 안 그려지는 걸 보완).
- `client.ts`, `admin/index.ts`, `common/index.ts`, `manage/index.ts` 같은 barrel/유틸 파일은 대상이 아님.

## 왜 이렇게 하나

두 규칙 모두 목적이 같다: **레이어를 넘나들 때 코드 추적 비용을 줄이는 것.** 백엔드는 URL/method로 Controller↔Service↔ServiceImpl을 잇고, 프론트는 `백엔드:` 줄로 API 함수↔Controller 메서드를 잇는다. 팀원이 늘어날수록 "이 API가 어디서 처리되는지" 찾는 데 드는 시간이 커지므로, 검색(grep) 한 번으로 대응 지점을 찾을 수 있게 하는 것이 핵심이다.

## 관련
- [[api-response-format]]
- [[api-client-usage]]
