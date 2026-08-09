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

레포지토리가 서비스별로 분리되어 있다 (각각 별도 git 저장소, `origin`은 `github.com/qKet/*`, GitHub organization 아래 소속). 로컬에서는 이 저장소들을 **형제 폴더로 나란히 클론**해서 하나의 워크스페이스 폴더(예: `Q_Ket/`) 아래 둔다 — 이 워크스페이스 폴더 자체는 git 저장소가 아니다.

- `frontend/` — Next.js/TypeScript
- `backend/` — Spring Boot, Gradle, MyBatis, 자체 `docker-compose.yml`(MySQL+Redis) 보유
- `Infra/` — Terraform + Kubernetes(EKS) 매니페스트 (운영/배포용 IaC — 로컬 개발과는 무관)
- `docs/` — 아키텍처/화면/ERD 이미지 및 문서
- `CLAUDE_LLM_WIKI/` — 이 위키 (Claude Code가 세션 시작 시 읽는 지식 베이스)

폴리레포이므로 **각 저장소는 다른 저장소 없이 독립적으로 실행 가능해야 한다** — `backend`는 `backend` 하나만 클론해도 뜨고, `frontend`도 마찬가지.

## 로컬 개발 환경

### 1. Backend — DB/Redis + 서버

```bash
cd backend
docker compose up -d      # MySQL 8 + Redis 컨테이너 기동
./gradlew bootRun         # http://localhost:8080
```

`application.yml`의 `DB_HOST`/`DB_PASSWORD`/`REDIS_HOST` 등은 전부 `docker-compose.yml`의 값과 일치하는 기본값이 박혀있어서 (`localhost`, `root`/`1234` 등) **`.env` 없이도 그냥 뜬다.** OAuth 소셜로그인(Google/Kakao/Naver)과 Toss 결제 연동만 안 되며, 이게 필요하면 `backend/.env`에 `GOOGLE_CLIENT_ID` 등을 채워야 한다(`load-env.ps1`이 Windows에서 이 `.env`를 읽어 환경변수로 주입).

> ⚠️ 예전엔 모노레포 루트에 있던 `docker-compose.yml`을 저장소 분리 때 `backend/`로 그대로 옮기면서, 볼륨 경로(`./backend/src/...`)가 안 고쳐진 채로 남아있던 적이 있었다 — `./src/main/resources/...`로 수정 완료. 자세한 내용은 [[docker-compose-stale-path-bug]] 참고.

### 2. Frontend

```bash
cd frontend
npm install
npm run dev                # http://localhost:3000
```

`lib/api/client.ts`가 `CLUSTER_IP` 환경변수 없으면 `http://localhost:8080`을 기본으로 쓰므로, 위에서 backend가 떠있으면 별도 설정 없이 바로 연결된다.

### 종료 / 초기화

```bash
docker compose down       # backend/ 안에서 — 컨테이너만 종료
docker compose down -v    # + 볼륨(데이터) 초기화, 시드로 복구됨
```

### 3. Infra (Terraform) — 처음 한 번만

로컬 개발엔 필요 없지만, 실제 AWS 인프라를 처음 적용하는 사람은 `workload`에서 **`terraform workspace new release`/`new prod`를 먼저 만들어야 한다** — 안 하면 `default` workspace라서 apply가 막힌다. 순서/이유는 [[terraform-apply-order]] 참고.

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
- [[docker-compose-stale-path-bug]]
- [[terraform-apply-order]]
