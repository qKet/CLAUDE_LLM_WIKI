---
title: API 응답 규격
category: conventions
tags: [backend, api]
created: 2026-08-06
updated: 2026-08-06
---

# API 응답 규격

## 규칙

- 새 컨트롤러는 그냥 데이터(List, DTO 등)를 `return`하면 됨 — `GlobalResponseAdvice`(`ResponseBodyAdvice`)가 자동으로 `{success, message, data, timestamp}`(`ApiResponse<T>`)로 감싼다.
- `Map`/`ApiResponse`/`ErrorResponse`/`String`을 직접 리턴하는 기존(레거시) 컨트롤러는 감싸지 않고, `timestamp` 필드만 주입해준다 (필드 순서: `success, timestamp, ...`).
- 실패는 `throw new BusinessException(ErrorCode.XXX)` (또는 커스텀 메시지 버전) — `ErrorCode` enum(`common.exception`)에 `(HttpStatus, code, message)` 정의. `GlobalExceptionHandler`(`RestControllerAdvice`)가 잡아서 `ErrorResponse{status, code, message, errors, timestamp}`로 응답.

## 왜 이렇게 하나

성공(`ResponseBodyAdvice`)과 실패(`RestControllerAdvice`) 처리를 분리하는 게 이 프로젝트 관례다. 새 컨트롤러를 작성하는 사람은 응답 포맷을 신경 쓸 필요 없이 데이터/예외만 다루면 되고, 포맷 통일은 두 어드바이스가 전담한다.

## 프론트엔드에서의 대응

프론트는 `apiFetch`가 `{success,data,timestamp}` 모양을 자동으로 `unwrap()`해서 `data`만 꺼내준다. 자세한 내용은 [[api-client-usage]] 참고.

## 관련
- [[api-client-usage]]
- [[comment-rules]]
