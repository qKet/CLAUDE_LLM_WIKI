---
title: 주석 규칙 (백엔드/프론트엔드)
category: conventions
tags: [backend, frontend, comment]
created: 2026-08-06
updated: 2026-08-09
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

### MyBatis Mapper XML: 쿼리 이름표

`resources/com/exam/**/mapper/*.xml`의 각 `<select>`/`<insert>`/`<update>`/`<delete>` 바로 위에 아래 형식의 이름표를 단다 (실제 예: `UserMapper.xml`).

```xml
<!--
     이름            :  findByEmail
     기능            :  이메일 중복 체크 — OAuthServiceImpl에서 기존 로컬/타 provider 계정과 이메일 충돌 확인용
     resultType     :  UserDTO
     parameterType  :  String
-->
<select id="findByEmail" resultType="UserDTO" parameterType="String">
```

- `이름`은 태그의 `id`와 동일하게 적는다(태그 attribute와 다르면 안 됨 — 예전에 `findByID`처럼 오타로 어긋난 적 있었음, 고칠 때마다 같이 맞출 것).
- `resultType`/`parameterType`은 **태그에 실제로 있는 속성만** 적는다. 없는 속성(예: `update`인데 `resultType`이 없는 경우)은 그 줄 자체를 뺀다 — 없는 값을 억지로 채우지 않는다.
- `기능`이 이 규칙의 핵심이다. **id/resultType/parameterType은 바로 아래 태그를 보면 이미 다 보이는 정보라 그대로 다시 타이핑하면 정보량이 0**이다. 대신 코드만 봐선 안 보이는 것 — 어느 Service/Controller가 어떤 흐름에서 이 쿼리를 호출하는지, 보안/데이터상 주의할 점(예: `pwd` 해시가 그대로 반환된다는 것과 호출부가 반드시 지워야 한다는 것) — 을 짧게 담는다.
- 파일 안의 SQL 하나에만 달아놓으면 왜 여기만 있나 헷갈리므로, **한 파일에 붙이면 그 파일의 모든 쿼리에 다 붙인다** (일부만 붙여놓는 건 스캔 용도를 오히려 해침).
- ⚠️ **대시(`-`)로 테두리를 꾸미면 안 됨**: `<!----------- ... ----------->` 처럼 하이픈을 2개 이상 연속으로 넣으면 XML 주석 스펙 위반(`--`는 주석 내부에 올 수 없음)이라 MyBatis가 파싱 실패로 **앱 기동 자체가 안 됨**. 실제로 8개 Mapper XML에 시도했다가 전부 파싱 에러 나서 되돌린 적 있음 — 구분선이 꼭 필요하면 `=`나 `*`(Java 쪽 `/***...*/`와 통일감 있음) 처럼 XML 주석에서 안전한 문자를 쓸 것.

## 왜 이렇게 하나

세 규칙 모두 목적이 같다: **레이어를 넘나들 때 코드 추적 비용을 줄이는 것.** 백엔드는 URL/method로 Controller↔Service↔ServiceImpl을 잇고, 프론트는 `백엔드:` 줄로 API 함수↔Controller 메서드를 잇고, MyBatis 쿼리 이름표는 `기능` 줄로 Service↔Mapper XML을 잇는다. 팀원이 늘어날수록 "이 API/쿼리가 어디서 처리되는지" 찾는 데 드는 시간이 커지므로, 검색(grep) 한 번으로 대응 지점을 찾을 수 있게 하는 것이 핵심이다.

쿼리 이름표는 처음엔 `id`/`resultType`/`parameterType`을 그대로 다시 타이핑하는 형태로 `UserMapper.xml`에 한 번 시도됐다가(태그 attribute와 어긋나는 오타까지 생김) 방치돼 있던 걸, "`기능` 줄에 코드에 없는 정보만 담기 + 파일 전체에 일관 적용"으로 정정한 것이다.

## 관련
- [[api-response-format]]
- [[api-client-usage]]
- [[mybatis-conventions]]
