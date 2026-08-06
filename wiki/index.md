# Index

Qket 프로젝트 팀 위키의 전체 페이지 목록. 새 페이지를 추가하면 여기 등록한다.

## onboarding/
- [[getting-started]] — 신규 팀원이 제일 먼저 읽는 페이지, 체크리스트 포함

## conventions/ — "우리는 이렇게 코딩한다"
- [[comment-rules]] — 백엔드 메서드 주석 / 프론트 API 함수 주석 형식
- [[api-response-format]] — `ApiResponse`/`ErrorResponse` 규격
- [[mybatis-conventions]] — XML 위치, type-alias 등록, nullable FK 가드 함정
- [[api-client-usage]] — `apiFetch` 사용 규칙
- [[admin-grid-pattern]] — 관리자 그리드 페이지 dirty-tracking 패턴
- [[css-file-structure]] — `styles/*.css` 역할별 분리 구조
- [[reusable-ui-components]] — `components/ui/` 컴포넌트 목록과 용도

## architecture/ — "시스템이 이렇게 동작한다"
- [[backend-layer-and-package-structure]] — Controller→Service→Mapper 4단 구조, `com.exam.*` 패키지
- [[auth-and-authorization]] — 세션/Role/IP 추적
- [[frontend-folder-structure]] — `lib/api`, `lib/data/types` 구조
- [[dynamic-menu-system]] — PROGRAMS/ROLE_PROGRAMS 기반 메뉴, 화면 노출 vs API 보안 구분
- [[db-schema-conventions]] — 감사 컬럼 6종

## decisions/ — "왜 이렇게 하기로 했나" (ADR)
- [[README]] — 아직 정식 기록 없음, 템플릿만 있는 상태

## troubleshooting/ — 실제 겪은 버그
- [[null-field-partial-update-bug]] — nullable FK 부분 업데이트 버그 (백/프론트 양쪽)
- [[utf8mb4-encoding-bug]] — docker exec MySQL 한글 이중 인코딩

## runbook/ — 반복 운영 절차
- [[db-schema-change]] — 로컬 DB 스키마 변경 절차 2가지

---

## 고아 페이지 / 미해결 항목

- 로컬 개발 환경 기동 절차([[getting-started]] 참고) — docker-compose 위치 등 아직 미기록
- [[decisions/README|decisions]] — 실제 ADR 0건, 팀 논의가 있을 때마다 채워야 함
