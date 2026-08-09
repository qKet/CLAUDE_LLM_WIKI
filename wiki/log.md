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

---

## [2026-08-06] Claude Code | troubleshooting | docker-compose.yml 볼륨 경로 버그 수정 + 온보딩 절차 확정

`backend/docker-compose.yml`이 예전 모노레포 루트 시절 상대경로(`./backend/src/main/resources/...`)를 그대로 가진 채 `backend/` 레포로 옮겨져 있던 걸 발견. 경로를 `./src/main/resources/...`로 수정하고 `docker compose config`로 실제 해석 경로를 확인.

- [[docker-compose-stale-path-bug]] 신설 — 증상/원인/해결/재발방지
- [[onboarding/getting-started]]의 TODO였던 "로컬 개발 환경 기동 절차"를 실제 명령어(`docker compose up -d`, `./gradlew bootRun`, `npm run dev`)로 채움 — `application.yml` 기본값이 compose 값과 일치해서 `.env` 없이도 기동됨을 확인
- `wiki/index.md`의 고아 페이지 항목에서 이 TODO 제거

---

## [2026-08-06] 이채영 | decision | CI 도구로 GitHub Actions 선택 (Jenkins 대신)

레포 분리 후 CI/CD 재설계 과정에서 Jenkins 기반 레퍼런스 아키텍처(빌드→ECR push→CD 레포 write-back→ArgoCD sync)를 검토. 구조 자체는 채택하되 CI 엔진은 Jenkins 대신 GitHub Actions로 결정 — "러너 직접 소유"와 "push 즉시 감지" 둘 다 GitHub Actions self-hosted runner + 기본 `on: push`로 동일하게 되고, Jenkins가 실제로 우위인 조건(대규모 빌드량, 멀티 VCS, 폐쇄망, 기존 Jenkins 자산)이 지금 팀엔 해당 없음을 확인.

- [[decisions/2026-08-06-ci-tool-github-actions]] 신설 (배경/결정/고려한 대안/트레이드오프)
- 확정된 다음 구조: backend/frontend 얇은 워크플로우 + `qKet/.github` 재사용 워크플로우 + `qKet/CD`(신규 GitOps 매니페스트 전용 레포) + ArgoCD. 실제 파일 작성은 아직 안 함 — index.md 고아 페이지 항목 참고.

---

## [2026-08-06] 이채영 | ingest | Terraform state 공유 구조(remote_state) 문서화

`workload/remote-state.tf`가 뭘 하는 파일인지 질의응답하며 나온 설명을 위키로 옮김 — S3 backend가 앱 리소스가 아니라 Terraform 자기 state 저장소라는 점, `platform`/`workload` state가 같은 버킷 안에 key로만 나뉜다는 점, `terraform_remote_state`가 state 전체가 아니라 `output`으로 공개된 값만 노출한다는 점(output = API 계약)까지 정리.

- [[terraform-remote-state]] 신설
- 미해결로 남긴 것: Infra/terraform이 오늘 크게 재편됐는데(모듈을 리소스 타입별로 쪼갬, `platform`/`workload` 2-root + workspace, ECR 재생성 결정 등) 이 부분 전체가 아직 위키에 없음 — index.md 고아 페이지 항목 참고.

---

## [2026-08-06] 이채영 | ingest | Terraform 모듈 재편 전체 문서화

오늘 진행한 Infra/terraform 대규모 재편(모듈을 `vpc`/`subnet`/`security_group`/`ec2`/`eks`/`rds`/`redis`/`storage`/`ecr`로 리소스 타입별 재편, `platform`/`workload` 2-root + terraform workspace 구조, root 파일을 `main.tf`/`providers.tf`로 통합) 전체를 위키로 정리.

- [[terraform-module-boundaries]] 신설 — 모듈 경계, security_group 범용화, ecr을 platform에 둔 이유
- [[terraform-platform-workload-split]] 신설 — platform/workload 구조, workspace_guard, env_config 패턴
- [[2026-08-06-terraform-module-restructure]] 신설 (ADR) — 재편 배경/결정/대안/트레이드오프
- [[2026-08-06-ecr-recreate-vs-import]] 신설 (ADR) — 기존 ECR(이미지 140개+) import 대신 삭제 후 재생성한 결정
- [[terraform-circular-module-dependency]] 신설 (troubleshooting) — 참고 레포 PAPERPLE-INFRA에서 발견한 vpc↔subnet 순환 참조
- [[terraform-apply-order]] 신설 (runbook) — platform→workload 적용 순서, workspace 생성 절차
- [[onboarding/getting-started]]에 terraform workspace 생성 필요성 링크 추가
- index.md 고아 페이지 항목에서 이 TODO 제거

---

## [2026-08-07] 이채영 | troubleshooting | GitHub Actions OIDC "Not authorized" — 근본 원인 미해결

`backend` 레포 CI의 OIDC AWS 인증이 계속 `Not authorized to perform sts:AssumeRoleWithWebIdentity`로 실패. `sub` 기본 형식 → qKet 조직이 subject claim 커스터마이징(불변 ID 포함)을 켜둔 걸 발견 → `repository`/`ref`/`job_workflow_ref`로 전환(AWS가 sub 또는 job_workflow_ref 필수라고 정책 자체를 거부해서 job_workflow_ref 추가) → 그래도 Not authorized, CloudTrail도 원인 안 드러남(뭉뚱그린 AccessDenied), SCP는 조회 권한 없어 확인 불가 → 결국 `sub` 하나만(ID는 와일드카드) 쓰니 바로 성공. **왜 커스텀 claim이 실패했는지는 끝내 못 밝힘.**

- [[troubleshooting/github-actions-oidc-not-authorized]] 신설 — 시행착오 전체 순서, 원인 미해결이라는 점과 지금 해법의 보안 트레이드오프(ID 와일드카드로 subject claim 커스터마이징 방어 무력화)까지 솔직하게 남김
- 미해결로 남긴 것: 근본 원인, ID를 정확한 값으로 좁히는 것 — index.md 고아 페이지 항목 참고

---

## [2026-08-07] 이채영 | troubleshooting | 위 이슈의 "남은 리스크"(ID 와일드카드) 해소

`repo:qKet@*/backend@*:...`처럼 와일드카드로 열어뒀던 불변 ID를, GitHub 공개 API(`curl api.github.com/repos/qKet/backend`)로 확인한 실제 값(`owner_id=313320752`, `backend repo_id=1323850932`, `frontend repo_id=1323797216`)으로 교체. `modules/github-actions-oidc`에 `github_owner_id`/`repository_id` 변수 추가.

- [[troubleshooting/github-actions-oidc-not-authorized]]의 "남은 리스크" 섹션을 "해결함"으로 갱신
- 근본 원인(왜 repository/ref/job_workflow_ref가 실패했는지)은 여전히 미해결 — 이건 그대로 열려있음

---

## [2026-08-07] 이채영 | ingest | 관리자 문의용 체크리스트 추가

내일 AWS Organizations 관리자(교육 계정 운영진)한테 SCP/RCP 여부 확인을 요청할 예정이라, 실패했던 정책 JSON·role ARN·CloudTrail 요청 ID를 그대로 들고 갈 수 있게 정리해서 문서에 추가.

- [[troubleshooting/github-actions-oidc-not-authorized]]에 "다음 단계 — 관리자에게 문의할 내용" 섹션 신설
- SCP/RCP가 원인이 아니라고 확인되면 repository/ref/job_workflow_ref 조합으로 재시도해보라는 것도 남겨둠

---

## [2026-08-07] 이채영 | troubleshooting | job_workflow_ref 단독 테스트도 실패 — 가설 강화, sub로 복구

`repository`/`ref`/`job_workflow_ref` 조합이 실패한 게 혹시 여러 claim을 한꺼번에 건 탓인지 확인하려고, `job_workflow_ref` 하나만 단독으로 조건에 걸어 격리 테스트 → 역시 `Not authorized`로 실패. `sub`/`aud` 두 claim 말고는 뭘 걸든(조합이든 단독이든) 이 계정에서는 전부 막힌다는 패턴이 더 뚜렷해짐 — 다른 조직에서는 `repository`/`ref` 커스텀 claim이 잘 동작한다고 널리 문서화돼 있어서, "AWS가 원래 sub/aud만 지원한다"는 일반적 한계가 아니라 이 계정에만 걸린 SCP/RCP 같은 설정일 가능성이 유력하다는 쪽으로 가설이 좁혀짐(확정은 아님).

- `modules/github-actions-oidc/main.tf`를 job_workflow_ref 단독 테스트 → 다시 `sub`+정확한 ID 조건으로 원상복귀, `terraform apply -target`으로 backend/frontend 둘 다 반영 확인 완료
- [[troubleshooting/github-actions-oidc-not-authorized]] "결론" 섹션에 5차 테스트 결과 반영, "다음 단계" 섹션의 실패 정책 예시에 job_workflow_ref 단독 버전(버전 B) 추가해서 관리자 문의 시 "조합이라 실패한 게 아니라 claim 자체가 막힌 것"이라는 근거를 명확히 함
- 여전히 미해결: SCP/RCP 여부는 관리자 확인 전까지 모름

---

## [2026-08-08] 이채영 | troubleshooting | "qKet이 옵션을 켰다"는 최초 추정 정정 — GitHub 전사 정책 변경(2026-04-23)이 원인

강사님이 본인도 OIDC 이슈를 겪었다고 언급 → 강사님 자료가 옛날 것일 수 있다는 질문을 계기로 재조사. GitHub가 2026-04-23에 "Immutable subject claims for GitHub Actions OIDC tokens"를 발표했고, **2026-07-15 이후 생성된 레포는 자동으로** `sub`에 불변 ID가 포함된 새 형식을 쓴다는 걸 확인(GitHub Changelog). `qKet/backend`·`qKet/frontend` 둘 다 실제 생성일이 2026-08-05로 컷오프 이후 — 즉 조직이 뭔가를 수동으로 "켜서" 생긴 게 아니라 최근에 만든 레포라 자동으로 새 형식이 적용된 것뿐이었음.

- [[troubleshooting/github-actions-oidc-not-authorized]] "1차" 섹션에 정정 블록 추가 — 최초 추정("옵션을 켰다")이 부정확했다는 것, 실제 원인(전사 정책+생성일 컷오프), 소스 링크, 강사님 사례와의 추정 연관성, 그리고 이게 어디까지나 `sub` 형식 문제(해결됨)에 대한 설명이지 아직 미해결인 `repository`/`ref`/`job_workflow_ref` claim 차단 문제(SCP/RCP 추정)와는 별개라는 점을 명시
- 문서를 코드/사실이 최신이 아니면 고친다는 원칙대로, 부정확했던 기존 서술을 삭제하지 않고 "정정" 블록으로 남겨둠(CLAUDE.md의 "모순은 삭제가 아니라 표시" 원칙)

---

## [2026-08-09] 이채영 | ingest | MyBatis Mapper XML "쿼리 이름표" 컨벤션 신설

`backend/.../auth/mapper/UserMapper.xml`의 `findById` 위에 있던 주석(`이름`/`id`/`resultType`/`parameterType`을 그대로 재타이핑 — 심지어 `findByID`로 오타까지 남)을 손보는 과정에서 시행착오를 거쳐 확정: id/resultType/parameterType처럼 태그만 봐도 아는 정보는 빼고, `기능` 줄에 실제 호출부(어느 Service/Controller, 어떤 흐름)와 코드만 봐선 안 보이는 주의사항(예: `pwd` 해시가 그대로 반환된다는 것)을 담는 형식으로 정착. `UserMapper.xml` 전체 쿼리(7개)에 일괄 적용해서 실제 예시로 남김.

- [[conventions/comment-rules]]에 "MyBatis Mapper XML: 쿼리 이름표" 섹션 신설 (기존 백엔드 Controller/Service, 프론트 API 함수 주석 규칙과 나란히)
- [[conventions/mybatis-conventions]]에 규칙 추가 + comment-rules로 상호 링크
- 아직 다른 Mapper XML(`SeatMapper`, `ReservationMapper`, `PerformanceMapper` 등)엔 소급 적용 안 됨 — comment-rules의 다른 규칙들처럼 "새로 만들거나 수정하는 쿼리부터" 적용 대상

---

## [2026-08-09] 이채영 | ingest | MyBatis 쿼리 이름표, 8개 Mapper XML 전체(63개 쿼리)로 소급 적용

바로 위 항목에서 "아직 소급 적용 안 됨"이라고 남겼던 걸, 사용자 요청으로 실제로 전체 소급 적용함 — `backend/src/main/resources/com/exam` 아래 Mapper XML 8개(`UserMapper`/`CategoryMapper`/`MenuMapper`/`ProgramMapper`/`PaymentMapper`/`PerformanceMapper`/`ReservationMapper`/`SeatMapper`) 전체 쿼리(총 63개)에 이름표를 달았다. 각 쿼리의 `기능` 줄은 추측이 아니라 실제로 Java Service/Controller를 grep해서 호출부를 확인하고 작성함.

- 태그 개수와 이름표 개수가 파일별로 정확히 일치하는지, XML이 여전히 well-formed인지 둘 다 스크립트로 검증 완료
- **부수적으로 발견한 것**: `SeatMapper.updateStatus`가 Java 코드 어디에서도 호출되지 않는 고아 메서드였음 — 실제 좌석 상태 변경은 `ReservationMapper.save`/`cancel`이 `RESERVATIONS` 테이블로 처리하고 있어서 이 메서드가 왜 남아있는지 불명. `SeatMapper.xml`의 해당 쿼리 위 주석에 그대로 남겨둠 — 삭제 여부는 팀 확인 필요
- [[conventions/mybatis-conventions]]/[[conventions/comment-rules]] 갱신은 안 함(형식 자체는 안 바뀜, 적용 범위만 넓어짐)

---

## [2026-08-09] 이채영 | troubleshooting | 이름표에 대시 테두리 시도 → XML 주석 규격 위반으로 8개 파일 전부 파싱 깨짐, 되돌림

이름표를 `<!----------- ... ----------->` 형태의 대시 테두리로 꾸미려고 8개 Mapper XML 전체(63개)에 일괄 적용했는데, XML 스펙상 주석 내부에 하이픈 2개 연속(`--`)이 오면 안 되는 규칙을 위반해서 전부 `not well-formed` 파싱 에러가 남 — 이 상태로 배포됐으면 MyBatis가 매퍼를 못 읽어서 백엔드가 기동 자체를 못 했을 것. `python xml.dom.minidom`으로 8개 파일 전부 파싱 실패 확인 후, 원래의 심플한 `<!-- -->` 형식으로 전부 되돌리고 재검증(파싱 OK + 태그/이름표 개수 63개 일치) 완료.

- [[conventions/comment-rules]]의 MyBatis 섹션에 ⚠️ 경고 추가 — 대시 테두리 금지, 필요하면 `=`나 `*` 사용할 것
- 최종 형식은 findById에 사용자가 직접 되돌려놓은 심플한 `<!-- 이름 : ... -->` 블록 형식으로 확정
