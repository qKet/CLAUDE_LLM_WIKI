---
title: MyBatis 컨벤션
category: conventions
tags: [backend, mybatis, mysql]
created: 2026-08-06
updated: 2026-08-09
---

# MyBatis 컨벤션

## 규칙

- XML 위치: `resources/com/exam/**/mapper/*.xml`, `namespace`는 Java Mapper 인터페이스의 FQN과 정확히 일치해야 함.
- `@Alias("XxxDTO")`가 붙은 DTO를 쓰는 패키지는 전부 `application.yml`의 `mybatis.type-aliases-package`에 등록돼 있어야 함 — DTO를 새 패키지로 옮기면 여기도 같이 고쳐야 한다.
- update 문에서 `<if test="field != null">` 가드는 **"값을 명시적으로 null로 지우는" 게 정상 케이스인 필드**(예: `parent_menu_id`, `program_id` 같은 nullable FK)에는 쓰면 안 됨 — 부분 요청(field 미포함)과 "null로 지우기"를 구분 못 해서, 의도치 않게 값이 사라짐. 이런 필드는 무조건 `SET field = #{field}`로 두고, **프론트가 항상 그 행의 전체 상태를 합쳐서 보내는 걸 전제**로 한다.
- 각 쿼리(`<select>`/`<insert>`/`<update>`/`<delete>`) 위에는 `이름`/`기능`/`resultType`/`parameterType` 형식의 이름표 주석을 단다 — 형식/예시는 [[comment-rules#MyBatis Mapper XML: 쿼리 이름표]] 참고.

## 왜 이렇게 하나

`<if test="field != null">`은 "필드가 안 왔을 때 건드리지 않는다"는 부분 업데이트 패턴엔 맞지만, nullable FK처럼 "null이 유효한 값"인 필드에 쓰면 `null`과 `undefined`(미전송)를 구분할 수 없는 MyBatis 특성 때문에 실제로 버그가 났었다. 이 이슈의 구체적인 재현 과정과 프론트엔드 쪽 대응(`getVal` 판단 로직)은 [[null-field-partial-update-bug]]에 정리되어 있다.

## 관련
- [[null-field-partial-update-bug]]
- [[admin-grid-pattern]]
- [[comment-rules]]
