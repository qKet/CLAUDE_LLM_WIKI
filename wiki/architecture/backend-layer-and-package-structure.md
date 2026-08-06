---
title: 백엔드 레이어/패키지 구조
category: architecture
tags: [backend]
created: 2026-08-06
updated: 2026-08-06
---

# 백엔드 레이어/패키지 구조

## 레이어 구조

```
Controller → Service(interface) → ServiceImpl → Mapper(interface) → XML
```

새 기능을 추가할 땐 이 4단 구조를 그대로 따른다. **Controller가 Mapper를 직접 호출하지 않는다.**

## 패키지 구조 (`com.exam.*`)

도메인별로 나뉘어 있고, 각 패키지 안에 `controller/dto/mapper/service/` 하위 폴더:

- `auth` — 로그인/세션/회원가입 (`UserController`, `UserDTO`)
- `admin` — 사용자·역할 관리(`AdminController`), 프로그램·메뉴 관리(`ProgramController`/`MenuController`, 관리자 전용 CRUD)
- `reservation` — 공연/좌석/예매(`PerformanceController`, `ReservationController`), 공연 관리(`AdminPerformanceController`, `/manage/*`)
- `common` — 도메인 무관 공통 기능: 예외 처리(`exception/`), 성공/에러 응답 어드바이스(`advice/`), 유틸(`WebUtil`), 파일 업로드(`CommonController`, `/common/upload`, 로그인만 하면 누구나 가능)
- `queue` — 예매 대기열

## 의도적으로 어색한 지점

`ProgramService`/`MenuService`/관련 DTO·Mapper는 `common`이 아니라 **`admin` 패키지**에 있다 — 프로그램/메뉴 관리가 관리자 전용 기능이라서다.

단, `CommonController`의 `GET /common/menus/my`(로그인 사용자 메뉴 트리 조회)는 `admin.service.MenuService`를 가져다 쓴다 — 이건 관리자 전용이 아니라 전체 사용자가 쓰는 기능이라 common→admin 의존 방향이 살짝 어색하지만 **의도적으로 그렇게 뒀다**. (신규 팀원이 "역방향 의존이다"라며 리팩터링하지 않도록 여기 남겨둠.)

## 관련
- [[api-response-format]]
- [[comment-rules]]
- [[dynamic-menu-system]]
