---
title: AdminAccessInterceptor가 context-path 때문에 전혀 매칭 안 되어 관리자 API 인가가 완전히 뚫려있던 치명적 버그
category: troubleshooting
tags: [security, authorization, interceptor, context-path, critical]
created: 2026-08-21
updated: 2026-08-21
---

# AdminAccessInterceptor가 context-path 때문에 매칭 안 되어 관리자 API 인가가 뚫려있던 버그

## 증상

로그인을 아예 안 했거나, 로그인은 했지만 일반 회원(roleId=1)인 사용자가 관리자 전용 API(카테고리/사용자/프로그램/메뉴/예약이력/공연관리, `/admin/**`·`/manage/**` 전체)를 **아무 제한 없이 호출 가능**했음. 조회는 물론 등록/수정/삭제까지 가능한 상태였음.

실제 검증(운영 배포본, `dev.jun979.click`):
```
GET https://dev.jun979.click/api/admin/categories   (로그인 세션 없이)
→ 200 OK, 실제 카테고리 데이터 전체 반환

대조군 — 컨트롤러 자체에 로그인 체크가 남아있는 엔드포인트:
GET https://dev.jun979.click/api/reservations/my   (로그인 세션 없이)
→ {"success":false,"message":"로그인이 필요합니다."}  (정상 차단)
```

## 원인

2026-08-18 리팩터링([[../log]] 참고)에서 관리자 컨트롤러 6개(`AdminController`, `AdminCategoryController`, `AdminReservationController`, `ProgramController`, `MenuController`, `AdminPerformanceController`)가 각자 메서드마다 갖고 있던 "로그인/관리자(roleId=3) 여부" 인라인 체크를 전부 삭제하고, `AdminAccessInterceptor` 하나가 `/admin/**`·`/manage/**` 경로 전체를 preHandle 단계에서 대신 걸러주는 구조로 통합함.

문제는 이 인터셉터 내부 구현:
```java
String path = request.getRequestURI();   // "/api/admin/categories" — context-path(/api) 포함
if (path.startsWith("/admin")) { ... }   // "/api/admin/..."은 "/admin"으로 시작 안 함 → 절대 매칭 안 됨
```

`server.servlet.context-path: /api`가 설정돼 있어서 `HttpServletRequest.getRequestURI()`는 **항상 context-path를 포함한 전체 경로**를 반환함(Servlet API 표준 동작). 그래서 `"/admin"`/`"/manage"` 접두어 매칭이 단 한 번도 성립하지 않았고, `preHandle()`이 두 분기 모두 안 타고 그대로 `return true`로 빠져서 **인가 검사 자체가 통째로 스킵**됐음.

**"인터셉터 하나로 통합"이 만든 단일 장애점(SPOF) 구조**라는 게 핵심 — 예전처럼 컨트롤러마다 개별 체크가 있었다면 인터셉터 버그가 나도 각 컨트롤러가 최후 방어선 역할을 했을 텐데, 지금은 인터셉터가 유일한 방어선이라 그게 뚫리면 관리자 API 전체가 동시에 뚫림.

## 해결

```diff
- String path = request.getRequestURI();   // context-path 포함
+ String path = request.getServletPath();  // context-path 제외 — DispatcherServlet 매핑 기준 경로
```

`getServletPath()`는 context-path를 뺀, 실제 서블릿 매핑 기준 경로(`/admin/categories`)를 반환하므로 접두어 매칭이 의도대로 동작함.

`backend` hotfix 브랜치로 `release`에 PR 올림: [qKet/backend#41](https://github.com/qKet/backend/pull/41) (머지는 팀원이 진행 예정, 이 문서 작성 시점 기준 아직 머지 전).

## 재발 방지

- **인가 체크를 여러 컨트롤러에서 인터셉터/필터 하나로 통합할 때는, 그 하나가 뚫리면 전체가 뚫린다는 걸 반드시 인지할 것** — 통합 자체는 "실수로 체크를 빠뜨리는 것"을 막아주는 좋은 패턴이지만, 유일한 방어선이 되는 만큼 그 방어선 자체의 테스트(특히 실제 배포 환경의 context-path 조건까지 포함한)가 더 중요해짐.
- **`context-path`가 있는 스프링 프로젝트에서 요청 경로를 다룰 땐 `getRequestURI()`(context-path 포함) vs `getServletPath()`(제외) 차이를 항상 의식할 것** — 로컬 개발 환경에서 `context-path`를 안 쓰거나 다르게 설정해서 테스트하면 이런 버그가 로컬에선 안 잡히고 배포 환경에서만 재현될 수 있음.
- 이런 종류의 "인가 우회"는 배포 후 **반드시 실제 배포 도메인에 인증 없이 관리자 API를 직접 호출해보는 통합 테스트**로 검증할 것 — 컨트롤러 단위 테스트만으로는 인터셉터/필터 체인 전체의 실제 동작을 못 잡아냄.

## 관련
- [[../conventions/api-response-format]]
- backend#41 (hotfix PR)
