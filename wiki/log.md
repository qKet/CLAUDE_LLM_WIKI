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
