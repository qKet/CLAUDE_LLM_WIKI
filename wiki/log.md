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

---

## [2026-08-10] MoonJunH | decision | 세션-대기열 Redis 인스턴스 공유 리스크 검토 및 페일오버 계획 확정

세션(Spring Session)과 대기열(`QueueService`)이 같은 ElastiCache Redis 인스턴스(`cache.t3.micro`, 싱글 노드)를 공유하는 현재 구조에서, 대기열 폭주 시 문제없을지 검토 요청을 받고 분석. 프론트 `QueueModal.tsx`의 3초 폴링 + `getStatus()`마다 `admitAvailableUsers()`가 같이 도는 구조를 근거로, 오픈런 시나리오(대기자 5,000명 기준 초당 약 8,000개 Redis 커맨드)에서 noisy neighbor(세션 응답 지연)와 메모리 압박(세션 eviction) 리스크가 실재함을 확인.

- 오토 페일오버는 운영(prod) 전환 시점에 켜기로 확정 (지금은 예산상 보류) — 다만 페일오버는 "장애 전파" 리스크만 해소하고 noisy neighbor·메모리 압박은 그대로 남는다는 점을 명확히 함
- 세션/대기열 물리 분리는 아직 미확정 — 부하테스트로 `t3.micro` 싱글 노드의 실제 임계치를 실측한 뒤 재검토하기로 함
- "분리하면 더 싸진다"는 오해를 정정 — 분리는 절감이 아니라 순수 추가 비용(노드 수 최대 4배)
- 논리 DB 분리(`SELECT`)는 비용 0원이지만 CPU/메모리 격리는 안 되는 임시방편으로 대안에만 기록
- [[decisions/2026-08-10-redis-session-queue-shared-instance-risk]] 신설 (배경/결정/고려한 대안/트레이드오프)
- [[architecture/auth-and-authorization]]에 상호 링크 추가

---

## [2026-08-10] 이채영 | troubleshooting | EKS provider 인증 문제(토큰 만료 + Unauthorized) 신규 문서화

`Infra/platform/providers.tf`의 kubernetes/helm/kubectl provider 설정에 남아있던 주석(왜 data source 대신 exec 방식인지, 왜 `--role-arn`이 꼭 필요한지)이 실제로는 문서화가 안 돼 있던 걸 확인 — 관련 troubleshooting 페이지가 없었음. 두 가지 실제로 겪었던 문제(① apply가 오래 걸리면 미리 받아둔 토큰이 만료돼서 뒷부분 리소스에서 인증 실패 ② `--role-arn` 없이 붙이면 Access Entry가 `cluster_admin` role한테만 등록돼 있어서 Unauthorized)를 신규 페이지로 정리.

- [[troubleshooting/eks-provider-auth]] 신설 — 증상 2개/원인/해결(exec 방식 + `--role-arn` 명시)/재발방지
- [[terraform-module-boundaries]]의 `cluster_admin` 공유 role 패턴과 연결
- index.md 갱신

**부수적으로 발견한 별개 문제**: 템플릿으로 참고하려고 [[troubleshooting/terraform-circular-module-dependency]]를 다시 읽다가, 문서엔 "NAT Gateway를 `modules/subnet`으로 옮겨서 해결했다"고 적혀있는데 실제 `Infra/platform/main.tf`는 NAT Gateway/라우팅을 `subnet` 모듈도 `vpc` 모듈도 아닌 **root(`platform/main.tf`) 레벨**에 두고 있는 걸 발견 — 코드와 문서가 어긋나 있었음(아마 설계 방향이 중간에 바뀌었는데 문서는 안 갱신됨). CLAUDE.md 원칙대로 기존 서술을 지우지 않고 ⚠️ 모순 블록으로 표시 + 실제 코드 위치와 근거를 남김.

---

## [2026-08-10] 이채영 | decision | 네임스페이스 생성을 workload/ArgoCD에서 platform으로 이전

`workload` apply 시 `namespaces "qket-release" not found` 에러 발생 — 원인은 네임스페이스를 ArgoCD(`Infra/kubernetes/{release,prod}/namespace_qKet.yaml`)가 관리하도록 이미 옮겨놨는데(이전 세션에서 `kubernetes_namespace.this`를 `terraform state rm`으로 뗌 — **이 마이그레이션 자체가 위키에 한 번도 기록된 적이 없었음**, 이번에 확인하며 발견), 그 YAML을 실제로 sync할 ArgoCD Application("infra-manifests")이 아직 안 만들어져서 새 클러스터엔 네임스페이스가 하나도 없는 상태였기 때문. 매번 수동 `kubectl apply`로 임시 처리하는 대신, `platform`(workspace 없이 한 번만 apply)이 `qket-release`/`qket-prod` 둘 다 `for_each`로 미리 만들어두도록 변경.

- `Infra/platform/main.tf`에 `resource "kubernetes_namespace" "qket"` 신설(`for_each`)
- `Infra/workload/main.tf`의 stale한 주석(주석 처리된 `kubernetes_namespace.this` 블록)을 정리하고 platform 쪽을 가리키도록 갱신
- [[runbook/terraform-apply-order]] 갱신: 스킴 경로 오타(`Infra/terraform/platform` → `Infra/platform`, 예전 모노레포 흔적) 수정, namespace 의존성 설명 추가, "아직 apply 안 됨"이라는 stale한 문구 제거(실제로는 이미 여러 번 apply/destroy 거친 상태)
- networkpolicy/ingress는 여전히 `Infra/kubernetes/*.yaml` + ArgoCD 관리 — 네임스페이스만 예외적으로 Terraform(platform)이 갖고 감
- **미해결로 남긴 것**: `Infra/kubernetes/{release,prod}/namespace_qKet.yaml` 파일이 이제 중복(platform이 이미 만듦) — 사용자 확인 결과 지금은 삭제하지 않고 남겨두고, ArgoCD "infra-manifests" Application을 실제로 만들 때 다시 정리하기로 함. 그 Application 자체도 여전히 미구현.

---

## [2026-08-10] 이채영 | decision | 이 위키 레포는 Claude Code가 직접 commit/push (팀원 전원 공통 규칙)

`frontend`/`backend`/`Infra`는 팀원이 직접 commit/push하는 게 원칙이지만, 이 위키 레포(`CLAUDE_LLM_WIKI`)만큼은 Claude Code가 갱신할 때마다 사용자 확인 없이 바로 commit/push까지 하기로 결정 — 위키는 "Claude Code가 유지보수하는 문서"라는 성격이라 다른 코드 레포와 다르게 취급. 처음엔 한 팀원 개인 세션에서만 적용되는 걸로 얘기가 나왔다가, "팀원들도 각각의 클로드가 직접 푸쉬하고 관리해야" 한다는 요청으로 **팀 전체 공통 규칙**으로 확정.

- `CLAUDE.md`(위키 운영 규칙)에 "다른 레포와 다른 점" 섹션 신설 + 세션 시작 체크리스트 6번 항목 갱신 — 어느 팀원의 세션이든 git으로 pull 받는 이 파일을 읽으면 동일하게 적용됨
- (참고) Claude Code의 로컬 메모리 기능에도 이 규칙을 기록해뒀지만, 그건 특정 기기/세션에 종속된 개인 기억이라 팀 전체엔 전파 안 됨 — 팀 규칙의 진짜 소스는 이 `CLAUDE.md`

---

## [2026-08-10] 이채영 | decision | workload의 rds/redis SG가 platform SG를 직접 참조하던 걸 CIDR 참조로 변경

`platform` destroy 중 `DependencyViolation`(bastion SG를 workload의 rds/redis SG가 아직 참조 중이라 삭제 거부)을 겪음. 처음엔 "destroy 순서를 맞추면 되는 문제"로 보였는데, 실제로는 "`workload`(RDS/Redis, 데이터라서 절대 안 지움)가 `platform`(자주 destroy/재생성)의 SG ID를 AWS 레벨에서 직접 참조하고 있다"는 더 근본적인 설계 문제였음 — 순서를 맞춰서 지워도 `platform` 재생성 시 SG ID가 바뀌어서 `workload`를 다시 apply해야 하는 문제가 남기 때문에, "platform만 독립적으로 껐다 켰다" 하려는 목표 자체와 안 맞았음.

`modules/subnet`에 `private_general_subnet_cidrs` output 추가 → `platform`이 이걸 노출 → `workload`의 rds/redis SG ingress가 SG ID 참조(`security_groups`) 대신 이 CIDR(`cidr_blocks`)로 접속을 허용하도록 변경. bastion과 EKS 노드가 같은 서브넷(private_general)에 있어서 규칙 2개(EKS SG 유래, bastion SG 유래)를 CIDR 기준 규칙 1개로 합칠 수 있었음.

- `modules/subnet/outputs.tf`, `platform/outputs.tf`, `workload/main.tf` 수정 (아직 apply 안 함 — 코드만 반영)
- [[architecture/terraform-platform-workload-split]]에 "workload가 platform의 SG를 참조하면 안 되는 이유" 섹션 신설, 스킴 경로 오타(`Infra/terraform/*`)도 같이 수정
- ⚠️ 남은 문제: `helm_release.argocd`/`kubernetes_namespace.qket`가 `platform` destroy 시 Access Entry 소실로 Unauthorized 나는 문제는 이것과 별개 원인이라 아직 미해결 — 제안된 해법(별도 root `cluster-bootstrap` 분리)도 미구현. index.md 고아 항목 참고.

---

## [2026-08-10] 이채영 | troubleshooting | EKS destroy 문제 종합 정리 — Layer 1(platform)/Layer 2(cluster-bootstrap) 분리로 결정

실제로 `platform` destroy를 돌리다 bastion SG `DependencyViolation`으로 중간에 멈추는 걸 겪음(NAT/라우팅/bastion 인스턴스/ECR/OIDC/ArgoCD/namespace/노드그룹은 지워졌는데 VPC/서브넷/EKS 클러스터/bastion SG만 남은 상태) — 이 과정에서 output이 이미 지워진 리소스(`helm_release.argocd` 등)를 참조하고 있어서 state의 output이 통째로 비어버렸고, 그 여파로 `workload`의 `terraform_remote_state` 읽기까지 "no attributes" 에러로 막힘.

이걸 계기로 지금까지 개별적으로 대응해온 문제들(증상1: kubernetes/helm provider Unauthorized, 증상2: workload↔platform SG 직접참조로 인한 DependencyViolation)이 사실 "**Layer 1(순수 AWS)과 Layer 2(K8s addon)를 한 state에 섞어놓은 것**"이라는 하나의 근본 원인에서 갈라져 나온 증상들이라는 걸 정리 — 사용자가 실제 운영 요구사항(workload는 절대 안 지움/데이터, platform은 매일 수동으로 켜고 끔/비용 절감, registry는 공유·불변, Ingress Controller·Grafana 같은 third-party는 따로 관리하고 싶음)을 구체화하면서 최종 구조를 4-root(`workload`/`registry`/`platform`/`cluster-bootstrap`)로 확정.

- [[troubleshooting/eks-destroy-layer-separation]] 신설 — 증상 3개(1·2는 실제로 겪음, 3번 Ingress Controller의 ALB/ENI 잔존 문제는 아직 안 겪었지만 같은 계열이라 미리 남김), 원인 표, Layer 분리 해법, **"현재 구현 상태" 섹션에 뭐가 코드로 됐고(SG CIDR 전환) 뭐가 아직 결정만 됐는지(cluster-bootstrap root 자체) 정직하게 구분**
- index.md 갱신 — 고아 항목을 이 문서 하나로 통합해서 가리키게 정리, Ingress Controller 재활성화 시 주의사항 추가
- 아직 미해결: `cluster-bootstrap` root 실제 생성, `registry` root 실제 생성, `workload`에 `prevent_destroy` 추가 — 전부 다음 작업으로 남음

---

## [2026-08-10] 이채영 | restructure | AWS 인프라 전체 destroy + root 디렉토리 이름 변경 (platform→infrastructure, workload→data)

정리하다 막힌 김에 사용자 결정으로 기존 AWS 인프라를 전부 지우고 새 구조로 다시 시작하기로 함. 실제로 진행한 것:

1. `data`(당시 workload) release workspace destroy — RDS/Redis/S3/CloudFront/IAM/SG 등 14개 리소스, 도중에 `module.storage`가 이미 null이 된 `infrastructure`의 OIDC provider 출력값 때문에 `replace()` 에러가 나서 destroy 전용 더미값으로 임시 우회 후 완료, 끝나고 원복
2. `infrastructure`(당시 platform) destroy 마무리 — 이전에 bastion SG DependencyViolation으로 멈춰있던 것을, `data`를 먼저 지운 뒤 재시도해서 완료(EKS 클러스터+VPC+서브넷 등 12개 리소스)
3. 이 과정에서 [[troubleshooting/eks-destroy-layer-separation]]에 증상 4(destroy 중단 시 output이 통째로 비어서 다른 root까지 막히는 문제)로 추가 기록
4. 사용자 요청으로 **root 디렉토리 이름을 전부 영어로 재정리**: `platform` → `infrastructure`, `workload` → `data`(구상 중이던 `cluster-bootstrap`/`registry`는 각각 `k8s-addon`/`registry`로 명명 확정, 아직 미생성). `git mv`로 이력 보존, backend.tf의 S3 key도 새 이름으로 갱신(state가 비어있던 시점이라 마이그레이션 없이 그냥 새 key로 재-init), `data.terraform_remote_state.platform` 참조도 전부 `infrastructure`로 리네임, `terraform validate` 둘 다 통과 확인

- [[architecture/terraform-platform-workload-split]], [[architecture/terraform-remote-state]], [[architecture/terraform-module-boundaries]], [[runbook/terraform-apply-order]], [[troubleshooting/eks-destroy-layer-separation]] 전부 새 이름 기준으로 갱신 — 파일명 자체는 안 바꿈(백링크가 많아서), 대신 각 문서 상단에 "이름이 바뀌었다"는 안내 블록 추가
- `Infra/backup/README.md`, `Infra/backup/platform/`(→`backup/infrastructure/`)도 같이 갱신 — 나중에 ALB Controller/ESO 재활성화할 때 이 문서 보고 따라 하면 새 이름 기준으로 맞게 동작하도록
- log.md의 과거 항목(위쪽)은 그 시점 실제 이름(`platform`/`workload`)이 맞으므로 안 고침 — CLAUDE.md의 "코드가 우선, 위키를 갱신" 원칙은 현재 상태를 설명하는 문서에 적용하는 것이지 과거 기록을 소급 수정하는 게 아님
- 결과: AWS 계정에 이 프로젝트 관련 리소스가 하나도 없는 완전히 빈 상태. 다음 apply부터 `infrastructure`/`data`라는 새 이름으로 처음 시작 — `k8s-addon`/`registry` root를 실제로 만들기 좋은 타이밍(마이그레이션 필요 없음)

---

## [2026-08-10] 이채영 | ingest | root 디렉토리에 apply 순서를 드러내는 숫자 접두사 추가

바로 위 항목에서 `infrastructure`/`data`로 이름만 바꿨는데, 사용자가 IDE에서 직접 `01_infrastructure`/`04_data`로 한 번 더 바꿔놓음 — Finder/IDE에서 정렬해도 apply 순서(infrastructure→k8s-addon→registry/data)와 그대로 일치하게 하려는 의도. 확인 결과 번호 체계 확정: `01_infrastructure`, `02_k8s-addon`(미생성), `03_registry`(미생성), `04_data`.

- git이 이 변경을 "예전 이름 삭제 + 새 이름 untracked"로 잘못 보고 있던 걸 `git add -A`로 다시 스테이징해서 rename으로 정상 인식시킴
- Infra 코드(모듈 주석, backup/README.md)와 위키 문서(관련 파일 전부) 안의 `Infra/infrastructure`, `Infra/data`, `infrastructure/main.tf` 같은 **경로** 참조를 전부 `01_infrastructure`/`04_data` 접두사 붙은 형태로 재수정
- S3 backend의 state key(`key = "infrastructure/terraform.tfstate"` 등)는 숫자 접두사 없이 그대로 둠 — 로컬 디렉토리 정렬용 접두사와 S3 키 네이밍은 무관하다고 판단
- `terraform validate`로 `01_infrastructure`/`04_data` 둘 다 정상 통과 재확인

---

## [2026-08-10] 이채영 | restructure | `02_k8s-addon` root 신설 — namespace/ArgoCD를 infrastructure에서 실제로 분리

[[troubleshooting/eks-destroy-layer-separation]]에서 계속 "결정만 됐고 미구현"이라고 남겨뒀던 걸 실제로 구현함. `01_infrastructure/main.tf`의 `kubernetes_namespace.qket`/`helm_release.argocd`(+ 관련 output `argocd_namespace`)를 새 root `02_k8s-addon`으로 옮기고, `01_infrastructure/providers.tf`에서 kubernetes/helm/kubectl provider를 전부 제거해서 `01_infrastructure`가 진짜 순수 AWS API 리소스 전용 root가 되게 함.

- `02_k8s-addon/`(backend.tf, remote-state.tf, providers.tf, variables.tf, main.tf, outputs.tf) 신규 작성 — provider 인증 패턴은 `01_infrastructure/providers.tf`/`04_data/providers.tf`와 동일(exec 방식 + `--role-arn`), `terraform_remote_state`로 `01_infrastructure`의 EKS 정보를 받아옴
- `01_infrastructure`/`02_k8s-addon`/`04_data` 셋 다 `terraform validate` 통과 확인(아직 `apply`는 안 함 — AWS엔 여전히 아무것도 없는 상태)
- [[runbook/terraform-apply-order]]를 2-root(infrastructure→data) 절차에서 **3-root(infrastructure→k8s-addon→data)** 절차로 전면 재작성
- [[architecture/terraform-platform-workload-split]], [[troubleshooting/eks-destroy-layer-separation]], index.md의 "미구현"/"결정만 됨" 표시를 전부 "구현 완료"로 갱신
- 남은 것: `03_registry`(ECR/github-actions-oidc 분리)는 여전히 미착수, Ingress Controller도 여전히 `backup/`에 보류 중

---

## [2026-08-10] 이채영 | decision | "01_infrastructure를 매일 껐다 켰다" → VPC/서브넷은 안 지우기로, `00_network` root 분리는 보류

`02_k8s-addon` 분리 직후 "04_data를 띄운 채로 01_infrastructure를 destroy해도 안전한가"를 확인하다가, `04_data`(RDS/Redis)가 `01_infrastructure`가 만든 서브넷 안에 물리적으로 떠있어서(SG CIDR 수정과 별개로) `01_infrastructure`를 통째로 destroy하면 여전히 위험하다는 걸 발견. `00_network`(VPC/서브넷을 또 분리하는) root를 제안했으나, 사용자가 "일이 점점 커진다"고 반응 — 재검토한 결과 **VPC/서브넷/라우팅테이블은 떠있어도 과금되지 않는다**는 걸 확인하고, 굳이 새 root로 안 쪼개도 됨을 확인.

- root 추가 없이, `01_infrastructure` 안에서 **비용 나는 리소스만 `-target`으로 골라 지우는** 방식으로 결정 — `module.eks`/`module.ec2`(bastion)/`aws_nat_gateway.this`/`aws_eip.nat`/`module.security_group`만 지우고 VPC/서브넷/라우팅/ECR/OIDC는 그대로 둠
- `Infra/down.sh`(저녁: k8s-addon 전체 destroy → infrastructure targeted destroy), `Infra/up.sh`(아침: infrastructure apply → k8s-addon apply) 스크립트 작성, 문법 체크 완료(실제 실행은 아직 안 함)
- [[runbook/daily-infrastructure-toggle]] 신설 — 배경(왜 새 root 대신 이 방식을 택했는지), 순서가 중요한 이유(k8s-addon 먼저 지워야 Unauthorized 안 남), 주의사항
- index.md 갱신
- **의사결정 과정 자체를 기록**: 처음엔 "구조적으로 제일 깔끔한" `00_network` 분리를 제안했다가, 사용자의 "계속 나누기만 하면 관리 부담이 커진다"는 현실적 피드백을 받고 더 단순한 대안으로 선회한 것 — 항상 "가장 깔끔한 구조"가 정답은 아니라는 사례로 남겨둠

---

## [2026-08-10] 이채영 | ingest | up.sh/down.sh 쉘 스크립트 폐기 → Infra/README.md로 전환

바로 위에서 만든 `Infra/up.sh`/`down.sh`를 사용자가 "sh 파일은 맥 사용자(나)만 편하게 쓴다 — 팀원 중 Windows 쓰는 사람 있으면 못 씀"이라는 이유로 폐기 요청. `terraform -chdir=...` 플래그를 쓰면 셸 종류를 안 타서 OS 상관없이 그대로 복붙 가능하다는 걸 활용해, 스크립트 대신 **명령어를 `Infra/README.md`에 직접 적어두는 방식**으로 전환.

- `Infra/up.sh`, `Infra/down.sh` 삭제
- `Infra/README.md` 신설 — root 구조 요약, 최초 적용 순서, 매일 아침/저녁 절차(`-chdir` 명령어), 위키 문서 링크 목록까지 포함
- [[runbook/daily-infrastructure-toggle]]도 스크립트 참조를 명령어 직접 기재로 갱신, 상단에 이 전환 이유를 명시

---

## [2026-08-10] 이채영 | ingest | `03_registry` root — 사용자가 이미 직접 구현해둔 걸 확인 + 문서 반영

`Infra/README.md`에 `03_registry` 최초 적용 순서를 추가하다가, 사용자가 "03 만들어져있을껄?"이라고 해서 확인해보니 실제로 이미 완성돼 있었음(`backend.tf`/`main.tf`/`outputs.tf`/`providers.tf`/`variables.tf` 전부) — `module.ecr`/`module.github_actions_oidc`를 `01_infrastructure`에서 깨끗하게 빼서 옮겨놨고, `01_infrastructure` 쪽에 중복/잔재 없음까지 확인. 덤으로 `Infra/argocd/qket-cd-app.yaml`(ArgoCD Application 매니페스트, `qKet/CD` 레포를 `qket-release`로 동기화)도 새로 생겨있는 걸 발견.

- `terraform validate`로 `03_registry` 정상 통과 확인, `01_infrastructure`도 재검증
- `Infra/README.md`의 "구상 단계, 아직 없음" 표시를 지우고 실제 apply 명령어로 교체, `argocd/qket-cd-app.yaml`도 설명 추가
- [[architecture/terraform-module-boundaries]]의 "ECR을 infrastructure에 둔 이유" 섹션을 "ECR을 registry에 둔 이유"로 갱신(제목부터 내용까지 — 더 이상 infrastructure에 없으므로)
- [[runbook/terraform-apply-order]]를 3-root에서 **4-root**(infrastructure→k8s-addon→registry/data, registry는 순서 무관) 절차로 재작성
- [[troubleshooting/eks-destroy-layer-separation]], index.md의 "registry 미착수" 표시를 전부 "구현 완료"로 갱신
- 남은 것: Ingress Controller만 여전히 `backup/`에 보류 중, `argocd/qket-cd-app.yaml`을 실제로 클러스터에 등록하는 것(App-of-Apps 구현)도 아직

---

## [2026-08-10] 이채영 | troubleshooting | 매일 아침 `04_data`도 다시 apply해야 하는 이유 발견

팀원이 `qKet/CD` 레포의 Helm 차트로 backend를 띄우려다 IRSA 권한 에러를 겪음 — 원인 진단 과정에서 `qket-backend` ServiceAccount(`04_data`가 만듦)가 `02_k8s-addon`이 관리하는 네임스페이스 **안에** 있다는 걸 재확인. 이어서 "그럼 `02_k8s-addon`을 매일 밤 destroy하면 그 안의 `04_data` 소유 오브젝트(ServiceAccount/ConfigMap)는 어떻게 되나"를 사용자가 직접 질문하면서 발견: **네임스페이스가 지워지면 Kubernetes가 그 안의 모든 오브젝트를 자동으로 cascade delete**하므로, `04_data`의 실제 AWS 리소스(RDS/Redis/S3)는 안전하지만 K8s 오브젝트 3개(ServiceAccount 1 + ConfigMap 2)는 매일 밤 같이 사라지고 `04_data`의 state와 drift가 생김.

이걸 `02_k8s-addon`으로 옮겨서 해결하려는 아이디어도 나왔으나, `04_data`의 값(RDS 포트/S3 버킷명 등)을 참조해야 해서 apply 순서상 순환 의존이 생겨 불가능하다는 것도 확인 — 대신 "아침에 `04_data`도 한 번 더 apply"(RDS/Redis는 안 건드리고 K8s 오브젝트 3개만 재생성, 비용/리스크 없음)로 대응하기로 함. 근본적 해결(ESO로 K8s 바깥에 연결 정보를 두는 것)은 나중 과제로 명시.

- `Infra/README.md`, [[runbook/daily-infrastructure-toggle]] 둘 다 아침 절차에 `04_data apply`(release/prod) 3번째 단계로 추가 + "왜 필요한가" 설명 섹션 신설
- Helm 차트 쪽 체크리스트도 주의사항에 추가: `serviceAccount.create: false` + `qket-backend` 참조 확인
- index.md 갱신

---

## [2026-08-10] 이채영 | decision | CD 레포 write-back 인증으로 GitHub App 선택 + 실제 구현 완료

`docs/cluadeDocs/ci-cd-cross-repo-auth.md`에 비교만 해두고 미뤄뒀던 PAT vs GitHub App 결정을 실제로 내림. PAT도 레포 하나로 스코프 좁히고 만료 설정하면 기술적으로 크게 위험하진 않다는 것까지 짚었지만, "장기 자격증명을 시크릿에 두기 싫다"는 사용자의 원칙적 이유로 GitHub App 선택 — AWS를 OIDC로 인증하는 것과 같은 맥락. 설정 과정에서 처음엔 개인 계정 소유로 잘못 생성됐다가(조직 설정 화면의 "My GitHub Apps" 링크가 개인 계정 앱 목록으로 연결되는 UI 함정), 삭제하고 `organizations/qKet/settings/apps/new`로 조직 소유 재생성.

- [[decisions/2026-08-10-cd-writeback-github-app]] 신설 — 배경/결정/고려한 대안(PAT)/트레이드오프/시행착오/구현 상세
- 실제 구현도 완료: `qket-ci-bot` App(qKet 소유, CD 레포에만 설치, Contents R/W만) 생성, `QKET_CI_APP_ID`/`QKET_CI_APP_PRIVATE_KEY` 시크릿을 backend/frontend 양쪽에 등록, `backend`/`frontend`의 `CI-release.yml`에 write-back 스텝(App 토큰 발급 → `qKet/CD` checkout → `values.yaml` image 값 sed 치환 → commit/push) 추가
- write-back 대상인 `CD/helm/values.yaml` 구조가 작업 도중 바뀌어서(이미지 줄 옆 주석이 없어짐) backend의 sed 패턴이 한 번 깨졌다가, "backend:"/"frontend:" 블록 range 매칭 방식으로 통일해서 재수정 — 실제 파일로 idempotency까지 테스트 완료
- index.md 갱신

---

## [2026-08-10] 이채영 | restructure | ALB Controller(AWS Load Balancer Controller) `02_k8s-addon`으로 재활성화

`CD` 레포의 Helm 차트가 Ingress를 만들어도 실제 ALB가 안 생기는 문제(Ingress Controller 자체가 없어서) — `backup/modules/alb-controller`에 보류돼 있던 걸 실제로 재활성화. [[troubleshooting/eks-destroy-layer-separation]]에서 이미 결정해둔 대로 `01_infrastructure`가 아니라 `02_k8s-addon`(Layer 2, K8s addon 전용)에 연결함.

- `git mv backup/modules/alb-controller modules/alb-controller`
- `02_k8s-addon`에 `module.alb_controller` 추가 — `01_infrastructure`의 remote_state(vpc_id, eks_cluster_name, oidc_provider_*)를 참조. `aws`/`http` provider가 이 root에 처음 필요해져서 `providers.tf`에 추가(기존엔 kubernetes/helm만 있었음), `project_name` 변수도 추가
- `01_infrastructure/outputs.tf`의 stale한 주석 처리된 `alb_controller_role_arn` output 삭제(이제 `02_k8s-addon/outputs.tf`에 있음), `backup/infrastructure/alb-controller.tf`(구 구조 참조하던 죽은 코드)도 삭제
- `terraform init`/`fmt`/`validate`로 `02_k8s-addon`/`01_infrastructure` 둘 다 정상 확인
- `backup/README.md`에서 ALB Controller 관련 내용 전부 제거(더 이상 보류 상태 아님), ESO 재활성화 안내만 남김 — ESO는 `04_data`의 RDS/Redis 값을 참조해야 해서 `02_k8s-addon`으로 못 옮기고 여전히 `04_data`에 둬야 한다는 것도 명시
- index.md/eks-destroy-layer-separation.md의 "Ingress Controller 미설치" 표시를 "완료"로 갱신
- Infra 레포 커밋은 평소처럼 사용자가 직접 함(git status에서 module rename으로 정상 인식되는 것까지만 확인)

---

## [2026-08-10] 이채영 | ingest | ExternalDNS 신규 추가 (`02_k8s-addon`)

ALB Controller로 ALB는 생기는데 `dev.jun979.click`/`app.jun979.click` 같은 도메인이 Route53에 자동으로 연결되진 않는다는 걸 계기로, ExternalDNS를 새로 추가함. ALB Controller와 완전히 같은 패턴(IRSA Role + `helm_release`)이라 `modules/external-dns`를 새로 만들어서 `02_k8s-addon`에 연결.

- 공유 AWS 계정(다른 팀들 Route53 존도 같은 계정에 있음)이라, `aws route53 list-hosted-zones`로 실제 `jun979.click` 존 ID(`Z0111999JD2RHOSHTM8A`)를 확인해서 `route53:ChangeResourceRecordSets` 권한을 **이 존 하나로만** 좁힘 — `List*` 계열만 Route53 특성상 전체 스코프(`"*"`) 필요(조회만 가능, 변경 불가). Helm 쪽에도 `domainFilters`로 같은 도메인을 한 번 더 걸어서 이중 방어
- `policy: upsert-only`로 설정 — Ingress가 지워져도 DNS 레코드는 자동으로 안 지움(공유 계정에서 실수 방지 우선, 필요해지면 `sync`로 전환 가능하다고 코드에 남겨둠)
- `terraform validate` 통과 확인
- [[troubleshooting/eks-destroy-layer-separation]] "현재 구현 상태"에 추가

---

## [2026-08-10] 이채영 | troubleshooting | CD Helm 차트 실동작 리뷰 — 문제 4가지 발견/수정

`CD/helm` 쪽이 write-back까지는 잘 되길래 "이대로 배포하면 진짜 돌아갈까?"를 코드 기준으로 직접 리뷰함. CI가 통과하고 이미지가 push되는 것과 별개로 런타임에 깨질 문제 4가지를 찾음:

1. `frontend-deployment.yaml`의 `CLUSTER_IP` env가 통째로 주석 처리 + 변수명도 `BACKEND_URL`로 오기 — API 호출 100% 실패 상황이었음. 주석 해제 + 이름 수정으로 해결
2. `Infra/argocd/qket-cd-app.yaml`의 `path: release`가 CD 레포 구조 변경(raw manifest → Helm) 후에도 안 고쳐져 있어서 ArgoCD sync가 100% 실패할 상태 — `path: helm`로 수정
3. `frontend/Dockerfile`이 여전히 3단계 멀티스테이지로 CI가 이미 끝낸 빌드를 중복으로 다시 돌리고 있었음(CI 주석은 "backend와 동일 패턴"이라고 돼 있었는데 실제론 안 맞았음) — backend와 동일한 single-stage로 재작성. 확인차 backend도 같이 점검했는데 backend는 처음부터 정상이었음(`EXPOSE` 포트 표기만 8090→8080으로 정리, 기능적 영향 없음)
4. backend가 실제로 필요로 하는 env var 전체(`application.yml`의 `${...}`)를 CD 차트의 `configMaps`/`secrets`와 대조 — `APP_BASE_URL`(소셜 로그인 리다이렉트 origin)이 안 채워져 있어서 `ingress.host` 재사용으로 해결(`helm template`로 렌더링 결과 확인). OAuth 3사 client-id/secret 6개 + `TOSS_SECRET_KEY`는 실제 발급받은 값이 없으면 채울 수 없어서 미해결로 남김(값 준비되면 ESO 패턴으로 새 Secret 추가하는 방향 제안)

- [[troubleshooting/cd-helm-chart-deploy-review]] 신설 — 4가지 문제 각각의 증상/원인/해결 + 공통 재발방지("CI 통과 ≠ 배포 시 동작")
- index.md 갱신 (고아 페이지 섹션에 OAuth/Toss 시크릿 미해결, prod ArgoCD Application 미존재 항목 추가)
- Infra/CD/backend/frontend 레포 커밋은 평소처럼 사용자가 직접 함

---

## [2026-08-11] 이채영 | decision | 모니터링 스택 설계 논의 (Prometheus/Grafana/Loki)

지표/DB상태/로그가 서로 다른 도구가 필요한 별개 문제라는 걸 먼저 정리하고, 각각 어떤 도구를 쓸지·어디에 배치할지 논의함. 실제 코드 구현 전, 설계 결정만 기록.

- `kube-prometheus-stack`(지표) + `Loki`(로그, ELK 대신 — 팀 규모에 안 맞음) + Grafana(CloudWatch 데이터소스로 RDS/Redis 상태까지 한 화면에서 봄)
- 흥미로운 삽질: 로그/지표 영구 저장소(S3/EBS)를 처음엔 `02_k8s-addon`에 두려다가, 사용자가 "밤에 지우면 사라지는거 아니야?"라고 바로 잡아냄 — `02_k8s-addon`은 매일 밤 destroy되는 root라 영구 저장소를 두면 안 됨. `04_data`는 workspace(release/prod)별로 두 번 생성되는 구조라 클러스터 전체가 공유하는 단일 저장소엔 안 맞고, 결국 이미 있던 "공유·불변·release/prod 구분 없음" 성격의 `03_registry`(ECR/OIDC)에 두기로 함 — 새 root를 안 만들고 기존 분류 체계를 재활용
- release/prod 로그를 한 버킷에 모아도 되는지도 논의 — Loki/Prometheus 둘 다 라벨 기반이라 물리적으로 한 곳에 저장돼도 조회 시 완전히 분리되므로 문제없음
- 1차 범위(이번에 같이): 클러스터 지표, 로그, CloudWatch DB 연동, 외부 헬스체크(Blackbox — 오늘 겪은 ExternalDNS 문제를 사람이 발견하기 전에 알림 왔을 것), ALB 에러율, 배포 annotation
- 2차 범위(나중): 비즈니스 지표(예매/결제), 비용 모니터링(오늘 겪은 고아 리소스 과금 감시), 분산 트레이싱
- [[decisions/2026-08-11-monitoring-stack-design]] 신설
- index.md 갱신 — 오늘 실제로 겪은 ALB destroy 순서/kubectl_manifest finalizer/webhook 레이스/ExternalDNS TXT 소유권 문제들이 아직 troubleshooting 문서로 안 남겨진 것도 고아 페이지 섹션에 표시해둠(추후 정리 필요)
