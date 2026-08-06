---
title: nullable FK 필드 부분 업데이트 버그
category: troubleshooting
tags: [backend, frontend, mybatis]
created: 2026-08-06
updated: 2026-08-06
---

# nullable FK 필드 부분 업데이트 버그

## 증상

관리자 그리드 화면(예: 메뉴 관리)에서 `parentMenuId` 같은 nullable FK 필드를 "값 있음 → null"로 바꿔 저장해도 반영되지 않거나, 반대로 건드리지 않은 필드가 의도치 않게 null로 지워지는 문제.

## 원인

두 레이어에서 각각 원인이 있었다.

**백엔드(MyBatis)**: update 문에서 `<if test="field != null">` 가드를 nullable FK에 썼다. 이 가드는 "필드가 안 왔으면 건드리지 않는다"는 의미인데, nullable FK는 "명시적으로 null로 지우는" 것도 정상 케이스라서, MyBatis 입장에서는 "안 보냄(부분 업데이트)"과 "null로 지움"을 구분할 수 없었다.

**프론트엔드**: 변경 여부 판단에 `getVal(...) ?? original`(nullish coalescing)을 썼다. `??`는 좌변이 `null`/`undefined`일 때 우변으로 폴백하므로, "사용자가 명시적으로 null로 바꿨다"와 "애초에 안 바꿨다(undefined)"를 구분하지 못해 화면에 반영이 안 되는 버그로 이어졌다.

## 해결

- **백엔드**: nullable FK 필드는 `<if test="field != null">` 가드를 쓰지 않고 무조건 `SET field = #{field}`로 둔다. 대신 **프론트가 항상 그 행의 전체 상태를 합쳐서 보낸다**는 걸 전제로 한다.
- **프론트엔드**: 값 판단을 `??`가 아니라 `field in changes[id]`로 한다 — "이 필드가 변경 목록에 키로 존재하는가"를 직접 확인해서 undefined와 명시적 null을 구분한다. 저장 시에도 변경된 필드만 보내지 않고 그 행의 최종 상태 전체를 합쳐서 보낸다.

## 재발 방지

- 새로운 관리자 그리드 화면을 만들 때는 [[admin-grid-pattern]] 컨벤션을 그대로 따른다.
- 새 nullable FK 컬럼을 추가할 때는 [[mybatis-conventions]]의 가드 규칙을 확인한다.

## 관련
- [[mybatis-conventions]]
- [[admin-grid-pattern]]
