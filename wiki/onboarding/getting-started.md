---
title: 신규 팀원 온보딩
category: onboarding
tags: [meta]
created: 2026-08-06
updated: 2026-08-06
---

# 신규 팀원 온보딩

## 프로젝트 개요

Qket — 공연 예매 시스템. Spring Boot(백엔드) + Next.js/TypeScript(프론트엔드).

## 저장소 구성

레포지토리가 서비스별로 분리되어 있다 (각각 별도 git 저장소, `origin`은 `github.com/qKet/*`):

- `frontend/` — Next.js/TypeScript
- `backend/` — Spring Boot, Gradle, MyBatis
- `Infra/` — Terraform + Kubernetes(EKS) 매니페스트
- `CLAUDE_LLM/` — 이 위키 (Claude Code가 세션 시작 시 읽는 지식 베이스)

## 로컬 개발 환경 (⚠️ 확인 필요)

백엔드는 MySQL 8 + Redis 컨테이너(`qket-mysql` 등)를 전제로 동작한다 ([[db-schema-change]] 참고). 다만 저장소 분리 이후 루트에 `docker-compose.yml`이 보이지 않아, **정확한 로컬 기동 방법(compose 파일 위치, 필요 환경 변수, 포트)은 아직 이 위키에 정리되어 있지 않다.**

> 이 섹션은 TODO다 — 처음 로컬 환경을 세팅하는 팀원이 겪은 과정을 그대로 여기에 채워 넣어달라. (예: 어느 저장소의 compose 파일을 쓰는지, `.env` 어디서 받는지, `load-env.ps1`(`backend/`)이 하는 역할 등)

## 코드를 보기 전에 먼저 읽을 페이지

1. [[backend-layer-and-package-structure]] — 백엔드가 어떻게 나뉘어 있는지
2. [[frontend-folder-structure]] — 프론트가 어떻게 나뉘어 있는지
3. [[api-response-format]] + [[api-client-usage]] — 프론트-백엔드가 어떻게 통신하는지
4. [[comment-rules]] — 새 코드를 작성할 때 지켜야 할 주석 규칙

## 첫 기여 전 체크리스트

- [ ] [[comment-rules]]에 정의된 주석 형식을 확인했다
- [ ] 새 API를 만든다면 [[api-response-format]]에 맞게 그냥 데이터를 return하면 된다는 걸 안다
- [ ] nullable FK 필드를 다루는 CRUD를 만든다면 [[null-field-partial-update-bug]]를 먼저 읽었다
- [ ] 관리자 그리드 화면을 만든다면 [[admin-grid-pattern]]을 따른다

## 관련
- [[db-schema-change]]
