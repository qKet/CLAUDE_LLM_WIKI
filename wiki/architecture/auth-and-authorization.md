---
title: 인증/인가
category: architecture
tags: [backend, security]
created: 2026-08-06
updated: 2026-08-06
---

# 인증/인가

## 세션

`HttpSession`(Redis-backed, Spring Session)에 `loginUser`(`UserDTO`) 저장. 각 컨트롤러가 `session.getAttribute("loginUser")`로 꺼내 씀.

## Role

`1=USER, 2=MANAGER, 3=ADMIN`. 각 컨트롤러에 `isAdmin()`/`isManagerOrAdmin()` 같은 헬퍼를 두고 **하드코딩된 role 체크로 API 자체를 지킴**.

> ⚠️ 이건 프론트의 메뉴 표시 여부([[dynamic-menu-system]] — PROGRAMS/ROLE_PROGRAMS 기반)와는 **별개의 보안 계층**이다. 프론트 메뉴는 화면 노출만 담당하고, 실제 API 접근 통제는 이 백엔드 role 체크가 전담한다. 둘을 하나로 합치려는 리팩터링은 하지 않는다 — 화면 노출 로직이 곧 보안이 되면, 메뉴 설정 실수가 API 권한 구멍으로 직결되기 때문.

## IP 추적

데이터를 바꾸는 모든 컨트롤러가 `HttpServletRequest`를 받아서 `WebUtil.getClientIp(request)`로 **매 요청마다 라이브로** IP를 뽑아 `ins_ip`/`upt_ip`에 넣는다. 세션에 캐싱된 값을 재사용하지 않는데, 세션 도중 IP가 바뀌어도 정확하게 추적하기 위함이다.

## 관련
- [[dynamic-menu-system]]
- [[db-schema-conventions]]
