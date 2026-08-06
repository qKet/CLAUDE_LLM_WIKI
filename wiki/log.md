# Log

작업 이력. append-only, 최신이 위로 가지 않고 아래에 계속 추가한다. 형식: `## [YYYY-MM-DD] 작성자 | 종류 | 제목`

종류: `restructure` | `ingest` | `decision` | `troubleshooting` | `lint`

---

## [2026-08-06] Claude Code | restructure | 개인 연구용 위키 → 팀 프로젝트 위키로 구조 전환

기존 `CLAUDE.md`가 개인 연구 지식베이스 스키마(concepts/tech/research)와 Qket 프로젝트 컨벤션(백엔드/프론트엔드 서술)을 한 파일에 같이 담고 있던 것을, 팀이 함께 쓰는 프로젝트 위키 구조로 재편.

- `wiki/conventions/`, `wiki/architecture/`, `wiki/decisions/`, `wiki/troubleshooting/`, `wiki/runbook/`, `wiki/onboarding/` 카테고리 신설
- 기존 `CLAUDE.md` 197~333줄(백엔드/프론트엔드 컨벤션 서술)을 15개 개별 페이지로 분리
- `CLAUDE.md`는 위키 운영 규칙(스키마 문서) 역할만 하도록 축소
- `wiki/index.md`, `wiki/log.md` 신설

미해결로 남긴 것: 로컬 개발 환경 기동 절차(docker-compose 위치 등)가 저장소 분리 이후 문서화되어 있지 않음 — [[onboarding/getting-started]]에 TODO로 표시.
