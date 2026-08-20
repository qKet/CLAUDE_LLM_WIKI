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

---

## [2026-08-11] 이채영 | troubleshooting | 실제 destroy 순서 사고 — IAM 손상 복구 + ALB/webhook 고아 리소스 정리

[[troubleshooting/eks-destroy-layer-separation]]이 이론으로만 경고했던 문제를 실전에서 그대로 겪음 — `01_infrastructure`를 `02_k8s-addon`보다 먼저 destroy하다가 `module.eks`의 cluster_admin IAM Role/Access Entry가 먼저 지워지면서 `02_k8s-addon`이 통째로 `Unauthorized`로 막힘. 복구하면서 새로운 패턴의 문제를 두 개 더 발견함: (1) 죽은 ALB Controller/ESO 파드가 정리 못 한 ALB/Target Group/Security Group이 AWS에 고아로 남는 문제, (2) 컨트롤러가 죽어도 그 컨트롤러가 등록한 admission webhook(`ValidatingWebhookConfiguration` 등)은 클러스터에 그대로 남아서 **이후 그 리소스 타입 전체의 finalizer 제거까지 영구히 막아버리는** 데드락 — ALB Controller(`aws-load-balancer-webhook`)와 ESO(`externalsecret-validate`/`secretstore-validate`) 둘 다에서 재현.

- 타겟 지정 재-apply(`-target=module.eks.aws_iam_role.cluster_admin` 등 4개)로 IAM 접근권 복구, AWS CLI로 고아 ALB/TG/SG 수동 삭제(같은 패턴이 그날 두 번 발생), 고아 webhook 설정 삭제 후 finalizer patch로 namespace terminating 완료, IAM Group에 남은 사용자 5명 제거 후 `module.eks` destroy 재실행
- [[troubleshooting/destroy-order-incident-and-webhook-orphans]] 신설 — 전체 증상/원인/복구 절차/재발방지
- [[troubleshooting/eks-destroy-layer-separation]]에 이 실제 사고를 상단 경고로 링크 추가
- `kubernetes_ingress_v1`(kubernetes provider, finalizer를 제대로 기다림)로 Ingress를 관리하고, Ingress에 `depends_on = [module.alb_controller]`를 걸어 destroy 순서를 강제하는 기존 코드 패턴이 왜 필요한지도 이 문서에서 근거로 남김

---

## [2026-08-11] 이채영 | ingest | `01_infrastructure`의 `try()` 방어 패턴 정리

`bastion` 보안그룹처럼 매일 밤 destroy 대상이라 "밤엔 없는 리소스"를 직접 인덱싱(`module.security_group.security_group_ids["bastion"]`)하던 코드 2곳(`outputs.tf`, `main.tf`의 `module "ec2"`)이 밤 시간대 `-refresh-only`/`plan`을 `Invalid index`로 하드-실패시키던 걸 `try(..., null)`로 감싸 해결한 기존 코드 수정을 문서화.

- [[architecture/terraform-platform-workload-split]]에 "`try()`로 nightly-off 상태의 `-refresh-only`/`plan`을 방어하는 패턴" 섹션 신설 — 앞으로 nightly-toggle 대상에 optional 리소스가 추가될 때 재사용 가능한 일반 해법으로 남김

---

## [2026-08-11] 이채영 | decision | OAuth/Toss 외부 API 키 Secrets Manager 관리 + 프론트 CI 키 전달 방식

[[troubleshooting/cd-helm-chart-deploy-review]]에서 미해결로 남았던 OAuth 3사/Toss 키 연결을 실제 발급값이 준비되면서 처리함. db-secrets/redis-secrets와 같은 ESO 패턴으로 Secrets Manager 새 시크릿 `external_api`(7개 키)를 만들고, 사람이 콘솔로 채운 값이 반복 `terraform apply`에 덮어써지지 않도록 `ignore_changes`로 보호. 프론트 빌드 타임에만 필요한 `NEXT_PUBLIC_TOSS_CLIENT_KEY`는 GitHub Secrets 복제 대신 프론트 CI가 이미 가진 AWS OIDC 인증으로 Secrets Manager에서 직접 fetch하도록 구현(사용자가 GitHub Secrets 사용을 명시적으로 거부: "깃 씨크릿은 쓰기싫은데").

- [[decisions/2026-08-11-external-api-secrets-manager]], [[decisions/2026-08-11-frontend-ci-toss-key-secrets-manager]] 신설
- `modules/eso`에 `external_api_keys` 변수/시크릿 추가, `modules/github-actions-oidc`에 frontend CI Role 전용 Secrets Manager 읽기 정책 추가(이름 접두사 와일드카드로 범위 제한)
- 이후 실제 결제 테스트 중 ESO 동기화 지연으로 K8s Secret에 stale한 값이 남아있던 문제를 겪음 — [[troubleshooting/payment-eso-secret-staleness]] 신설(원인 오인했던 과정도 같이 기록: 페어링 불일치가 아니라 동기화 지연이었음)
- [[troubleshooting/cd-helm-chart-deploy-review]]의 미해결 표시를 해결됨으로 갱신, index.md 갱신

---

## [2026-08-11] 이채영 | decision | `/api` 라우팅을 Next.js `rewrites()`에서 ALB path 라우팅으로 이전

[[troubleshooting/cd-helm-chart-deploy-review]]의 문제 1 해결책이었던 `next.config.js`의 `rewrites()` 프록시 방식이 실제 프로덕션 버그를 냈음 — Next.js `rewrites()`는 빌드 타임에 값을 정적으로 얼리는데, 이게 컨테이너 이미지 재사용 배포 모델과 안 맞아서 배포 시점 값이 반영 안 되는 문제로 나타남. `/api/*` 라우팅 책임을 프론트 컨테이너에서 완전히 빼고 ALB가 path 기반으로 backend/frontend Ingress에 직접 라우팅하도록 변경.

- Ingress를 `app_ingress_backend`(path `/api`)/`app_ingress_frontend`(path `/`)로 분리 — healthcheck-path가 서로 다른 것도 분리 이유(annotation이 Ingress 오브젝트 단위 적용이라 하나로는 둘 다 못 만족)
- `group.name` 공유로 물리 ALB는 하나 유지
- [[decisions/2026-08-11-frontend-api-routing-alb-not-rewrites]] 신설
- [[troubleshooting/cd-helm-chart-deploy-review]]의 문제 1 섹션에 "⚠️ 모순" 표시로 이 변경을 링크(기존 "해결"이 더 이상 실제 배포 방식이 아님을 명시)
- 미해결로 남김: 로컬 개발(`npm run dev`) 환경엔 ALB가 없어서 `/api` 프록시 방식이 배포 환경과 갈라짐 — 온보딩 문서 반영 필요

---

## [2026-08-11] 이채영 | decision + troubleshooting | 모니터링 실제 구현(Grafana+CloudWatch) + Grafana/ArgoCD 공유 IP제한 ALB

[[decisions/2026-08-11-monitoring-stack-design]]에서 논의만 해뒀던 설계를 실제로 구현. 단, 범위를 좁혀서 Loki(로그)는 이번엔 빼고 클러스터 지표+DB CloudWatch 연동+Grafana 노출까지만 진행. Grafana 노출 방식도 논의 때 정한 "포트포워딩만"에서 "IP 허용목록 Ingress(공개 도메인)"로 바뀜 — 실사용 편의성 때문. ArgoCD UI와 하나의 ALB를 공유하는 구조로 합침(처음엔 도구별로 분리했다가, IP 정책이 어차피 팀 전체로 동일해서 하나로 합치는 게 관리 포인트가 적다고 판단).

- `modules/monitoring` 신설(kube-prometheus-stack helm_release + Grafana IRSA/CloudWatch 읽기전용 정책)
- `02_k8s-addon/admin-ingress.tf` 신설 — `grafana.jun979.click`/`cd.jun979.click` 전용 ACM 인증서(DNS 검증 자동화), `group.name = "qket-admin"` 공유, `inbound-cidrs`로 팀 IP만 허용, 서로 다른 healthcheck-path(로그인 무관 200 고정 엔드포인트로 일부러 선택)
- ALB Controller가 `group.name`을 바꿔도 옛 IngressGroup(ALB/TG)을 자동으로 안 치운다는 것도 실제로 겪음(그룹을 분리→합치는 과정에서) — 옛 ALB 2개 수동 정리
- [[decisions/2026-08-11-monitoring-stack-design]]에 "구현 결과" 섹션 추가(논의 당시 기록은 보존, 실제로 다르게 한 부분만 표로 정리), [[architecture/admin-ingress-shared-alb]], [[troubleshooting/alb-ingressgroup-orphan-on-rename]] 신설
- index.md 갱신

---

## [2026-08-11] 이채영 | troubleshooting | backend CI에 MySQL/Redis 서비스 컨테이너 추가

`DemoApplicationTests.contextLoads()`가 CI 러너에 MySQL/Redis가 없어서 `RedisConnectionException`으로 실패하던 걸 GitHub Actions `services:` 블록(빈 MySQL 8.0/Redis 7 컨테이너, `application.yml` 로컬 기본값과 동일한 값)으로 해결. 실제 RDS/ElastiCache에 CI를 연결하는 방향도 검토했으나 네트워크 격리/데이터 오염 리스크/nightly-toggle 인프라 상태와의 결합 문제로 기각(사용자도 동의: "그냥 빈깡통으로 연결이 되는지 안되는지 테스트한다고 ㅇㅇ").

- [[troubleshooting/backend-ci-missing-service-containers]] 신설

---

## [2026-08-11] 이채영 | decision | dev/Grafana/CD VPN 접근제어 — 논의만 하고 팀 상의 위해 보류

IP 허용목록(`inbound-cidrs`)의 확장성 한계에서 시작해 VPN 도입을 논의. AWS Client VPN(비용 문제로 기각) vs WireGuard(bastion에 설치, 기울었음), 지금의 공개+IP제한 ALB vs internal ALB로 완전 비공개 전환(기울었음) 두 축을 검토했으나, 팀 전체 워크플로우에 영향을 주는 결정이라 사용자가 "팀원들이랑 더 상의를 해보고 결정해야할 사안"이라며 명시적으로 논의를 중단함.

- [[decisions/2026-08-11-vpn-access-control-paused]] 신설 — 결정이 아니라 논의 스냅샷임을 문서 상단에 명시, 팀 결정 없이 임의로 재개하지 말 것을 남김
- index.md 고아 페이지 섹션에 추가

---

## [2026-08-12] 이채영 | ingest | 이메일 발송 인프라(SQS+Lambda+SES) 구현

팀원의 SES/SQS/Lambda 이메일 인증번호 발송 제안을 실제로 구현. 새 root(`05_messaging`) 대신 `04_data`에 `module "messaging"`으로 접붙이고(release/prod마다 큐/Lambda 분리 필요라는 점이 RDS/Redis와 같은 성격), SES 도메인 인증(계정당 1회만 가능)만 `03_registry`로 분리. Lambda 코드는 외부 빌드 스텝이 없는 순수 Node.js라 CI/S3 업로드 대신 Terraform `archive_file`로 로컬 zip 배포(처음엔 CI-built S3 zip을 권장했으나 코드 실체를 보고 단순화).

- `modules/messaging` 신설(`sqs.tf`/`lambda.tf`/`iam.tf` — SQS 하나·ARN 하나로 좁힌 IAM, SES 발송권한도 도메인 identity ARN 하나로 제한), `03_registry/ses.tf` 신설(domain identity+DKIM)
- Lambda 코드(`lambda-src/index.mjs`)를 팀원이 작성한 실제 코드로 교체 — 처음엔 화면 캡처가 잘려서 재구성한 placeholder였는데, 실제 코드를 받아보니 재구성 버전과 로직이 거의 동일했음
- `03_registry` 첫 apply에서 SES가 이 코드 이전에 이미 콘솔 등으로 인증돼 있던 상태와 충돌(DKIM CNAME 3개 "already exists") — `terraform import`로 기존 레코드를 state에 편입해서 해결
- Lambda 배포 중 `AWS_REGION`을 예약 env var로 직접 설정하려다 막힘(런타임 자동 주입값이라 직접 설정 불가) — `environment` 블록에서 제거
- 사용자가 `runtime`을 `nodejs24.x`로 시도했으나 AWS provider(`~> 5.0`, 실제 5.100.0)가 v6.21.0 미만이라 client-side validation에서 막힘 확인 — 4개 root 전체가 얽힌 메이저 업그레이드라 사용자가 직접 진행하기로 함("내가 바꿔볼게")
- 예매완료/결제내역 이메일 등 이후 이메일 종류를 늘릴 때는 새 큐/Lambda 대신 기존 것에 `type` 필드로 분기 추가하는 방향을 권장(인프라 리소스 추가 불필요) — 사용자가 직접 코드를 작성하겠다고 해서 실제 코드 변경은 안 함, 방식만 설명
- [[architecture/messaging-infrastructure]], [[decisions/2026-08-12-messaging-infra-placement]], [[troubleshooting/ses-dkim-preexisting-records-import]], [[troubleshooting/lambda-env-var-and-runtime-version-gotchas]] 신설
- index.md 갱신 — 콘솔에 남은 예전 `qket-email-verification-lambda` 정리 필요, end-to-end 발송 테스트 미완료, prod workspace 미적용을 고아 페이지 섹션에 추가

---

## [2026-08-12] 이채영 | decision | 팀원이 공유한 "예매확정 알림 파이프라인" 아키텍처 노트 반영 (SQS vs SNS, EventBridge Scheduler 보류 근거)

팀원이 claude.ai 아티팩트로 공유한 설계 문서("예매확정 알림 파이프라인: SQS·Lambda·SES")를 위키로 이관. 문서가 제안한 구조 부분(Lambda 전용 신규 레포, Infra `05_messaging` 신규 root)은 이미 확정된 [[decisions/2026-08-12-messaging-infra-placement]]와 달라서 **참고만 하고 따르지 않기로 함** — 대신 "왜 SQS→Lambda→SES 조합을 골랐는가"라는 아키텍처 선택 근거(동기 대신 비동기로 분리한 이유, SQS를 낀 이유, SNS 대신 SQS를 고른 이유, EventBridge Scheduler를 보류한 이유)만 뽑아서 정리함.

- [[decisions/2026-08-12-sqs-lambda-ses-notification-pipeline]] 신설 — SQS vs SNS 비교표(pull/push, 단일소비자/fan-out, 재시도 방식), EventBridge Scheduler는 시간 기반 트리거 요구사항이 확정되기 전까진 안 만드는 게 맞다는 근거 포함
- [[architecture/messaging-infrastructure]]에 SES 발신 도메인(`jun979.click`) 인증 현황 표 추가(DKIM/프로덕션 액세스 완료, DMARC는 Terraform 밖에서 설정된 상태라 나중에 SES DKIM 때와 같은 import 충돌 가능성 있음, **SPF는 아직 미등록**) + "의도적으로 특이한 지점"에 "Lambda가 아직 이메일 인증번호만 처리하고 예매확정 알림은 설계만 있다"는 현재 상태 명시
- index.md 갱신 — SPF 미등록, EventBridge Scheduler 용도 확인 필요, Lambda의 예매확정 알림 미반영을 고아 페이지 섹션에 추가

---

## [2026-08-12] Claude Code | troubleshooting | 모니터링 스택 실제 구현 + Grafana-AMP 연동 미해결

[[decisions/2026-08-11-monitoring-stack-design]]의 1차 범위 중 다수를 실제로 구현하고 검증함. 지표 영구저장은 문서상 방침(EBS)과 다르게 **AMP로 방향을 바꿈** — 이유는 아래 참고.

**구현 완료**
1. `module.monitoring` 실제 apply (kube-prometheus-stack)
2. backend API 지표 ServiceMonitor 추가(release만) — Service에 `spec.selector`만 있고 `metadata.labels`가 없어서 Prometheus가 discover는 해도 relabel 단계에서 계속 drop하던 문제를 겪음. ServiceMonitor의 selector는 Service의 selector가 아니라 Service 자체의 라벨을 본다는 걸 몰라서 헤맴 — `CD` 레포의 Service 템플릿에 라벨 추가로 해결
3. 대시보드를 git(ConfigMap)에 저장해서 EKS 재생성에도 자동 복구되게 함(`sidecar.dashboards`) — 실제로 Pod 재시작 후 자동 복구 확인함(단, Grafana 계정/비밀번호 같은 SQLite 데이터는 여전히 초기화됨 — 별개 문제)
4. **지표 영구저장 방식을 EBS에서 AMP로 변경**: EBS(PVC)는 StorageClass `ReclaimPolicy: Delete`라 EKS destroy 시 PVC와 함께 볼륨도 삭제돼서 "클러스터 재생성에도 데이터 유지"라는 목표 자체를 못 푸는 걸 확인함(Pod 재시작에는 강하지만 클러스터 재생성에는 무력) — AMP는 EKS와 완전히 분리된 관리형 서비스라 이 문제 자체가 없음. `01_infrastructure`에 `aws_prometheus_workspace` + IRSA 신설, `remoteWrite` 검증(`prometheus_remote_storage_samples_total` 33만 건+, 실패 0건)
5. 적용 중 겪은 공통 패턴: **Helm 값만 바뀌고 Pod 스펙(ServiceAccount annotation 등)이 안 바뀌면 Helm이 Pod를 자동 재시작 안 시킴** — IRSA 자격증명은 Pod 생성 시점에만 주입되므로, ServiceAccount annotation 변경 후 Pod를 수동으로 delete해서 재생성해야 실제로 반영됨. env var 변경은 Pod spec 자체가 바뀌어서 자동 롤아웃됨(대조적).

**미해결로 남김**: [[troubleshooting/grafana-amp-datasource-missing-auth-token]] — Grafana가 AMP에 remoteWrite는 잘 하는데, 그 데이터를 Grafana에서 직접 조회하는 것 자체가 안 됨. AMP API/IAM 권한은 수동 boto3 서명 테스트로 정상 확인했고, IRSA와 access key 둘 다 동일하게 실패하는 것까지 확인해서 자격증명 문제가 아니라 `grafana-amazonprometheus-datasource` 플러그인(v3.1.0) 자체의 버그로 결론. 사용자와 상의 후 "일단 여기서 멈추고 이슈로 기록"하기로 함 — 데이터는 AMP에 안전하게 쌓이고 있어서 급한 문제는 아님.

- [[troubleshooting/grafana-amp-datasource-missing-auth-token]] 신설
- index.md 갱신 (monitoring-stack-design 항목을 "구현 상당 부분 완료 + 남은 미해결" 상태로 업데이트, troubleshooting 목록에 신규 문서 추가)
- `Infra` 레포 `feature/nyj` 브랜치에 커밋 — ServiceMonitor, 대시보드 ConfigMap, AMP remoteWrite 코드까지 push 완료(PR 리뷰 대기). Grafana-AMP 데이터소스 연동 코드(`additionalDataSources`)는 남겨뒀지만 아직 실제로는 안 됨

---

## [2026-08-12] 이채영 | ingest | 제출용 "기획서" 위키에 추가 (Obsidian PDF 내보내기용)

`docs/cluadeDocs/기획서.md`로 먼저 작성했던 프로젝트 기획서(동기/배경, 2차 프로젝트 회고, 고도화 목표, 대기열 실측 데이터, 아키텍처, 실제 고도화 내역)를 사용자가 PDF로 뽑아야 한다며 위키로 옮겨달라고 요청. Obsidian이 이 위키 vault를 열어 쓰고 있다는 점(`.obsidian/` 확인됨)을 근거로, 별도 변환 도구 없이 **Obsidian 자체 "내보내기 → PDF"**로 바로 뽑을 수 있게 순수 마크다운(Mermaid 다이어그램 포함, Obsidian이 네이티브 렌더링)으로 작성.

- 표준 6개 카테고리(conventions/architecture/decisions/troubleshooting/runbook/onboarding) 중 어디에도 안 맞는 문서라서, 위키 루트에 `기획서.md`로 두고 index.md에 "산출물(제출용 문서)" 섹션을 새로 만들어 예외로 등록
- 내용 자체(동기/배경/기술적 근거)는 `docs/cluadeDocs/기획서.md`와 동일 — 포맷만 위키/Obsidian PDF 내보내기에 맞게 조정
- 팀원 이름/역할/일정처럼 위키에 없는 정보는 `[ ]`로 표시해서 사용자가 직접 채우게 남김

---

## [2026-08-12] Claude Code | troubleshooting | Grafana-AMP 미해결 이슈 2차 조사 — 여전히 미해결, 업스트림 버그로 확정

[[troubleshooting/grafana-amp-datasource-missing-auth-token]]를 사용자 요청으로 다시 팜. systematic-debugging 절차대로 새 가설 2개를 세우고 실제 클러스터에서 직접 검증함.

- 가설 1(URL 끝 슬래시로 인한 이중 슬래시 경로 문제)을 `trimsuffix()`로 Terraform 코드까지 고쳐서 실제 적용·Pod 재시작까지 해봤으나 기각 — queryData는 그대로 실패, 오히려 기존에 통과하던 checkHealth까지 깨짐. 원래 값으로 원복함(현재 `Infra` 작업 트리는 커밋 상태와 동일, 실제 변경 없음)
- 결정적 증거: SigV4 서명을 아예 안 붙인 요청이 플러그인의 실패와 **완전히 동일한 에러**를 냄 — 자격증명 문제가 아니라 애초에 `Authorization` 헤더 자체가 안 붙는 버그라는 게 사실상 확정됨
- authType `ec2_iam_role` 옵션(Terraform 안 거치고 Grafana API로 임시 데이터소스 만들어 빠르게 검증)도 동일하게 실패 — `default`/`keys`/`ec2_iam_role` 3가지 인증 방식 전부 실패로 확인됨
- GitHub 업스트림에서 정확히 같은 증상의 미해결 이슈([grafana-amazonprometheus-datasource#640](https://github.com/grafana/grafana-amazonprometheus-datasource/issues/640)) 발견 — Helm provisioning 환경의 알려진 버그이고, 유일한 workaround("Save and Test" 수동 클릭)는 우리처럼 readOnly provisioning인 경우 애초에 쓸 수 없음
- 결론: 코드/설정으로 해결 불가능한 명확한 업스트림 버그. 더 이상 파는 것보다 플러그인 버전 교체나 SigV4 프록시 sidecar 같은 우회가 필요 — 문서에 다음 시도 후보로 남기고 이번엔 여기서 멈춤
- [[troubleshooting/grafana-amp-datasource-missing-auth-token]] "추가 조사" 섹션과 결론 갱신

---

## [2026-08-13] Claude Code | troubleshooting | 부하테스트/모니터링 기반 트러블슈팅 3건 정리 (KEDA·metrics-server, HikariCP 커넥션 폭풍, CPU 스로틀링 vs 오토스케일링)

k6 부하테스트를 실제로 돌리면서 Grafana/Prometheus/CloudWatch/`kubectl` 지표로 잡아낸 문제 3건을 문서화. 전부 "겉으로 보이는 증상"과 "실제 원인"이 다른 층위에 있던 케이스라 진단 과정 자체를 상세히 남김.

1. **KEDA가 CPU 트리거를 걸어도 전혀 스케일 안 함** — `kubectl get hpa`가 `<unknown>/70%`로 멈춰있었는데, `kubectl describe hpa`의 Condition까지 봐야 `metrics-server` 자체가 클러스터에 없다는 게 드러남. 설치 후 replica가 4→8까지 실제로 늘어나는 것까지 검증.
2. **부하테스트만 돌리면 관련 없는 ArgoCD/Grafana까지 같이 멈춤** — RDS CloudWatch 지표(CPU 9%, 연결 51~61)는 멀쩡해서 처음엔 원인불명이었으나, HikariCP 풀 과다 설정(40×8replica=320)으로 인한 "커넥션 생성" 자체의 CPU 비용이 버스터블(t3.medium) 노드의 CPU 크레딧을 노드 단위로 고갈시켜 같은 노드의 다른 파드까지 굶긴 것으로 확인. 풀 사이즈 축소로 해결.
3. **CPU 스로틀링 2단계 진단** — 1차는 CPU limit이 작아서(250m/1) 상향(500m/2)으로 해결. 2차는 자가진단용으로 새로 추가한 Grafana 패널(pod별 CPU 스로틀링, 노드 CPU 사용률)로 재관찰한 결과, limit 상향 후에도 순간 스로틀링은 남아있는데 KEDA는 "정상적으로" 안 늘리고 있는 걸 발견 — CPU 트리거가 limit이 아니라 request 대비 평균 사용률을 보기 때문(순간 버스트는 5분 평균에 안 잡힘)이라는 구조를 확인. replica 증설이 아니라 개별 파드 limit 추가 상향을 권고(아직 미적용, 사용자 판단 대기).

- [[troubleshooting/keda-scaling-missing-metrics-server]], [[troubleshooting/hikaricp-connection-storm-load-test]], [[troubleshooting/backend-cpu-throttling-and-scaling-load-test]] 신설
- [[architecture/keda-autoscaling]] 신설 — KEDA 구조를 troubleshooting 문서들과 별개로 참조 가능하게 정리(엔진/규칙 분리 배치, ArgoCD `ignoreDifferences`로 replicas 충돌 방지 등)
- index.md 갱신 — troubleshooting/architecture 목록 추가, 고아 페이지 섹션에 "CPU limit 2차 상향 미적용", "PR #15 아직 dev로 merge 안 됨" 추가

---

## [2026-08-18] Claude Code | troubleshooting + decision | frontend CPU 스로틀링 원인 규명(CFS vs JVM) + 대용량 트래픽 용량 분석

frontend에 KEDA를 적용하는 과정에서 Grafana "CPU 스로틀링" 패널에 유휴 부하에서도 지속되는 스로틀링을 발견, Prometheus 직접 질의로 실측 확인 후 원인 규명과 처방을 정리. 이어서 사용자의 "대용량 트래픽 대비 지금 스펙이 안전한가" 질문에 클러스터 실측 기반으로 답변한 내용도 위키에 정리.

- [[troubleshooting/frontend-cpu-throttling-cfs-quota-vs-jvm-tradeoff]] 신설 — CFS 100ms 쿼터 + Node.js 보조 스레드(GC/libuv/JIT) 버스트가 원인. frontend는 CPU limit 완전 제거(request만 유지)로 해결(`CD/helm/values.yaml`, `values-prod.yaml` 반영). backend는 JVM이 cgroup CPU 쿼터를 읽어 `Runtime.availableProcessors()`/스레드풀 크기를 자동 사이징하는 컨테이너 인지형 런타임이라(`UseContainerSupport`, `ActiveProcessorCount` 명시 오버라이드 없음 확인) 같은 처방을 안 씀 — limit 유지가 오히려 안전하다는 근거까지 문서화
- [[decisions/2026-08-18-capacity-planning-large-traffic-readiness]] 신설 — `kubectl describe node`/`top nodes`로 실측한 결과 노드 CPU가 baseline에서 이미 60%대이고, 결정적으로 **cluster-autoscaler/Karpenter가 리포 어디에도 설치돼 있지 않아 `node_max_size=3`이 무의미**하다는 걸 확인(KEDA가 파드를 늘려도 노드가 안 늘어남 → Pending). RDS 커넥션 여유(85 중 80 예약)와 버스터블 CPU 크레딧 리스크, Redis 단일노드 소용량도 재확인. 상태는 논의중 — 분석/우선순위 권고(오토스케일러 > RDS 상향 > Redis > 장시간 재측정)까지만, 실제 조치는 미착수
- [[architecture/keda-autoscaling]] 갱신 — 제목/본문을 backend 전용에서 backend+frontend 공통 구조로 확장, ArgoCD `ignoreDifferences`는 서비스별로 각각 등록해야 한다는 점과 두 서비스의 CPU limit 정책이 다르다는 점 추가
- [[troubleshooting/backend-cpu-throttling-and-scaling-load-test]]에 2026-08-18 실측 결과(유휴~저부하 기준 backend 스로틀링 정확히 0) 교차 반영, 상호 링크 추가
- index.md 갱신 — decisions/troubleshooting/architecture 항목 추가, 고아 페이지 섹션에 cluster-autoscaler 부재를 신규 미해결 항목으로 추가

---

## [2026-08-18] Claude Code | decision + architecture | cluster-autoscaler 설치, RDS release 상향 — 대용량 트래픽 용량 분석 후속 조치

같은 날 앞서 작성한 [[decisions/2026-08-18-capacity-planning-large-traffic-readiness]]의 권고를 사용자와 함께 실제로 실행. Cluster Autoscaler vs Karpenter 중 프로젝트 규모(인스턴스 타입 1종, 노드 최대 3대)에 맞춰 Cluster Autoscaler로 결정하고 바로 구현·검증까지 완료.

- [[architecture/cluster-autoscaler]] 신설 — `Infra/modules/addons/cluster-autoscaler`(IRSA+helm, 다른 addon과 동일 패턴) 구조 문서화. EKS 관리형 노드그룹의 ASG에 autodiscovery 태그가 AWS에 의해 이미 자동으로 붙어있음을 실측 확인(노드그룹 Terraform 코드는 전혀 안 건드림), IAM 쓰기 권한은 `k8s.io/cluster-autoscaler/<클러스터명>=owned` 태그 조건으로 스코프(같은 계정에 team1-eks/doro-erp-dev 등 다른 팀 클러스터도 있어서). 설치 직후 IRSA trust policy 전파 지연으로 파드가 1회 크래시 후 자동 재시작되는 것도 정상 동작으로 확인·문서화(daily-infrastructure-toggle 루틴에서 매번 재현 예상)
- RDS release 인스턴스를 `db.t3.micro` → `db.t3.medium`으로 상향(`04_data/main.tf` 수정 → `terraform apply`). `aws_db_instance`에 `apply_immediately`가 없어서 기본적으로 다음 유지보수 기간까지 미뤄지는 것을 확인, 사용자 확인 후 `aws rds modify-db-instance --apply-immediately`로 즉시 적용까지 완료(실제 전환 검증함). prod(`db.t3.small`)는 보류
- Redis는 사용자가 명시적으로 보류 결정 — 나중에 대기열 기능 구현 시 세션/대기열 물리 분리(기존 [[decisions/2026-08-10-redis-session-queue-shared-instance-risk]]의 대안 A)로 갈 가능성이 있어 그때 같이 정하는 게 낫다는 판단. 해당 문서에 2026-08-18 노트로 반영
- [[decisions/2026-08-18-capacity-planning-large-traffic-readiness]] 상태를 "논의중"에서 "부분 구현 완료"로 갱신, "구현 결과" 섹션 추가(권고 1·2 완료, 3 보류, 4 미착수)
- index.md 갱신 — architecture/decisions 목록, 고아 페이지 섹션 반영

---

## [2026-08-18] Claude Code | troubleshooting | Grafana-AMP 데이터소스 인증 버그 해결

6일간 미해결이던 [[troubleshooting/grafana-amp-datasource-missing-auth-token]] 문제 해결. 그동안 GitHub Issue #640에 다른 사용자가 남긴 코멘트를 놓치고 있었음 — 서버 레벨 `sigv4_auth_enabled`만 켜고 데이터소스 `jsonData.sigV4Auth` 필드를 빠뜨렸던 게 원인. 두 설정을 같이 켜니 즉시 해결(`terraform apply -target=module.monitoring`, Grafana pod 재시작 후 `/api/ds/query`로 200 확인, 5일 전 데이터까지 정상 조회됨).

- 위키 문서 "미해결" → "해결됨"으로 갱신, 원인/재발방지 섹션 추가
- index.md 두 곳(troubleshooting 목록, monitoring-stack-design 항목) 갱신
- `Infra` `feature/nyj`에 커밋+push 완료(`modules/addons/monitoring/main.tf`) — apply 전 `dev`가 15개 커밋 앞서있는 걸 발견해서 먼저 merge(클러스터 오토스케일러, KEDA, RDS 인스턴스 변경, `node_instance_types` t3.xlarge 반영 등 포함), merge 중 `admin-ingress.tf`에 콤마 누락 문법 오류 하나 발견해서 같이 수정

---

## [2026-08-18] Claude Code | troubleshooting | AMP 전환 후 넓은 시간 범위 조회 시 예전 데이터 빈 값 문제 해결

[[troubleshooting/grafana-amp-datasource-missing-auth-token]] 해결 직후 대시보드 패널 14개를 전부 AMP 데이터소스로 전환했는데, 넓은 시간 범위(Last 7 days)로 보면 최근 몇 시간만 나오고 예전 데이터(8/13)는 빈 값으로 보이는 새 문제를 발견. `rate(...[5m])`처럼 고정된 짧은 lookback 윈도우가 원인 — 클러스터가 간헐적으로만 켜져있어 예전 샘플이 듬성듬성한데, 넓게 볼 때 쿼리 시점 간격이 `[5m]`보다 커서 대부분 샘플을 놓침. `$__rate_interval`(Grafana 내장 변수, 현재 시간 범위에 맞춰 윈도우 자동 조절)로 교체해서 해결 — 7일 범위 조회 시 데이터 포인트 전부 채워지는 것 확인.

- [[troubleshooting/grafana-amp-rate-interval-sparse-historical-data]] 신설
- index.md 갱신 (troubleshooting 목록에 신규 문서 추가)
- `Infra` `feature/nyj`에 커밋+push, PR #22(`feature/nyj → dev`)에 자동 반영됨(대시보드 AMP 전환 PR과 같은 브랜치라 별도 PR 안 만들고 기존 PR에 커밋 추가)

---

## [2026-08-18] Claude Code | troubleshooting | 만 명 오픈런 부하테스트 — k6 클라이언트 함정 3개 + ALB 헬스체크 연쇄 장애 발견 (하루 마무리 정리)

`decisions/2026-08-18-capacity-planning-large-traffic-readiness`의 4번 권고(목표 규모 재측정)를 실제로 실행. `Infra/loadtest/`에 `open_run_10000_no_queue.js`/`open_run_10000_with_queue.js` 신설(대기열 포함/미포함 2버전, 실제 백엔드 컨트롤러 코드를 읽어서 홈→로그인→상세→(대기열)→좌석조회→예매 풀 여정 재현). 500명 → 만 명까지 단계적으로 올리면서 문제 4개를 순서대로 발견·해결(3개)/발견만(1개, 최심각).

- [[troubleshooting/loadtest-10000-open-run-cascading-failures]] 신설 — 이번 회차의 핵심 문서:
  1. `constant-vus`(반복형) executor가 VU 수를 부풀림(500 VU가 실제로는 이터레이션 4742회) 발견 → `per-vu-iterations`+`iterations:1`로 교체해서 "N명이 각자 한 번씩"을 정확히 재현
  2. macOS가 TLS 인증서 검증을 블로킹 syscall로 처리해서 VU 수천 개 몰리면 k6 프로세스 자체가 Go 스레드 상한(1만)으로 크래시 — `insecureSkipTLSVerify: true`로 회피
  3. VU 1만 개 진짜 동시 접속 시 ALB 자체가 TLS negotiation 단계에서 리셋(CloudWatch `ClientTLSNegotiationErrorCount` 분당 6~8천 건 확인, `RejectedConnectionCount`/`TargetConnectionErrorCount`는 0이라 ALB 자체 문제로 특정) — 각 VU 시작 전 `0~RAMP_SECONDS`초 랜덤 대기로 커넥션 개설을 분산시켜 해결
  4. 🔴 **위 3개를 다 걷어내고 나서 발견한 진짜 서버 문제**: 부하가 심해지면 파드가 헬스체크(backend `/api/actuator/health`, frontend는 무거운 SSR `/`)에도 제때 응답 못 해서 ALB가 로테이션에서 제외 → 남은 파드에 더 몰림 → 도미노로 전부 제외 → `HealthyHostCount=0`(완전 다운)까지 CloudWatch로 실제 재현 확인(20:17 backend/frontend 동시에 0). **미해결** — 다음 세션 최우선 과제로 frontend 헬스체크 경로를 무거운 `/` 대신 전용 엔드포인트로 분리하는 것을 권고
  5. 부수 발견: ArgoCD `repo-server`가 리소스 request 없음(`BestEffort` QoS)이라 같은 부하 중 자기 헬스체크에도 응답 못 해 재시작당함 — ArgoCD UI가 "connection refused" 뱉은 원인. **미해결**
- [[troubleshooting/2026-08-18-config-build-bugs]] 신설 — 같은 날 만난 독립적 버그 3건 묶음(2026-08-13 패턴과 동일): `03_registry` output이 존재 안 하는 `module.ses` 참조하던 버그, `notification-config`라는 실체 없는 ConfigMap을 Helm values가 참조해서 backend가 `CreateContainerConfigError`로 못 뜨던 버그(app-config에 이미 필요값 있어서 참조 제거로 해결), backend Dockerfile이 `eclipse-temurin:21-jre`(OS 미고정) → Ubuntu Noble 전환으로 `useradd --uid 1000` 충돌하던 버그(`-jammy` 태그 고정으로 해결, 실제로 `docker run`으로 UID 1000 비어있음 검증)
- [[decisions/2026-08-18-capacity-planning-large-traffic-readiness]] 갱신 — 4번 권고 실행 결과 반영, backend KEDA `maxReplicas` 8→12·`limits.cpu` 2→3 상향 기록, ALB 연쇄 장애를 이 문서의 분석 프레임(노드/DB/Redis 스펙)으로는 못 잡아내는 새로운 리스크 층위로 명시, "다음 세션에서 이어갈 것" 체크리스트 추가
- [[architecture/cluster-autoscaler]] 갱신 — 500명 테스트로 3번째 노드 자동 증설, KEDA 4→8 스케일아웃 전부 실전 검증됐음을 기록. 동시에 오토스케일러만으로는 순간 폭증(thundering herd)을 못 막는다는 한계도 명시
- [[troubleshooting/frontend-cpu-throttling-cfs-quota-vs-jvm-tradeoff]] 갱신 — CPU limit 제거 수정이 한동안 CD 레포에 미배포 상태였다가(로컬 커밋만 있고 push 안 됨) 뒤늦게 배포 완료된 경위 기록 — "코드를 고쳤다고 끝이 아니라 실제 배포까지 확인해야 한다"는 교훈 남김
- index.md 갱신 — troubleshooting 목록, 고아 페이지 섹션(ALB 헬스체크 연쇄 장애·ArgoCD repo-server를 신규 미해결 최우선 항목으로 등록, backend CPU limit 상향 항목은 해결로 갱신)

**오늘 하루 전체 요약(내일 이어갈 사람을 위해)**: cluster-autoscaler 설치→RDS release 상향→만 명 부하테스트까지 큰 흐름은 다 진행됐고, KEDA/오토스케일러 조합은 실전 검증까지 끝났음. 남은 것 중 제일 급한 건 **ALB 헬스체크 연쇄 장애**(frontend 헬스체크 경로 분리가 직접 해법)와 **ArgoCD repo-server 리소스 request 추가** 둘 다 원인은 찾았고 고치는 방법도 정해졌지만 아직 코드 반영 전. 그다음은 `open_run_10000_with_queue.js`로 대기열이 이 연쇄 장애를 실제로 막아주는지 재검증하는 것.

---

## [2026-08-19] Claude Code | troubleshooting | ALB 연쇄 장애 재현 + 애플리케이션 레벨 원인 2건 특정, GitHub 이슈 3건 등록

전날 미해결로 남겨뒀던 [[troubleshooting/loadtest-10000-open-run-cascading-failures]]의 "문제 4"(ALB 헬스체크 연쇄 장애)가 오늘 부하테스트 재개하자마자 그대로 재현됨(backend 타겟그룹 healthy 8/draining 6/unhealthy 4). 여기에 frontend가 OOMKilled(메모리 limit 256Mi 부족)되는 신규 증상도 같이 관찰됨.

- 사용자가 "이거 인프라 문제야 애플리케이션 문제야?"라고 물어서 처음엔 "인프라(캐파시티+설정 실수+오토스케일러 반응 지연)가 대부분"이라고 답했는데, 되짚어 코드를 실제로 확인해보니 그 답이 불완전했음을 인정하고 재조사:
  - **backend**: `application.yml`에 `management.server.port` 지정이 없어서 actuator 헬스체크가 일반 요청과 같은 워커 스레드풀을 씀 — 부하 몰리면 헬스체크도 같이 타임아웃나서 연쇄 장애를 코드가 직접 조장하고 있었음
  - **frontend**: 홈(`app/page.tsx`)과 상세(`app/events/[performanceId]/page.tsx`) 둘 다 `fetch(..., { cache: "no-store" })`로 캐싱이 전부 꺼져있어서, 안 바뀌는 데이터(카테고리/공연목록/상세)도 매 요청마다 backend 재왕복+SSR 재렌더링 — 홈이 부하테스트에서 제일 먼저 무너졌던(10% 성공률) 이유를 코드로 설명 가능해짐
- [[troubleshooting/loadtest-10000-open-run-cascading-failures]] "2026-08-19 후속" 섹션 추가 — 위 2건 상세 + "이날 문제 중 어디까지가 인프라/코드인가" 표 정리
- 사용자가 "팀원들과 협업할 수 있게 정리만 해달라(당장 고치지 말고)"고 요청 → GitHub 이슈 3건 등록(각각 원인/증거/제안 포함):
  - [qKet/frontend#27](https://github.com/qKet/frontend/issues/27) SSR 캐싱 추가
  - [qKet/frontend#28](https://github.com/qKet/frontend/issues/28) 헬스체크 전용 엔드포인트 신설
  - [qKet/backend#29](https://github.com/qKet/backend/issues/29) actuator 헬스체크 포트 분리
- 그 외 오늘 처리한 것(비교적 사소):
  - `02_k8s-addon`에서 새로 생긴 root 간 CRD 순서 문제 발견 — `argocd-notifications-eso.tf`(ArgoCD 알림 이메일 기능 추가하면서 신설, 커밋 `4e7a965`)가 `04_data`의 ESO가 설치하는 `external-secrets.io/v1` CRD를 참조하는데, 확립된 apply 순서(`infrastructure→k8s-addon→data`)와 반대 방향 의존이라 매일 아침 재적용마다 재현될 것으로 예상됨 — 오늘은 `04_data`의 `module.eso`를 먼저 targeted apply해서 우회. **아직 위키 문서화 안 함, 사용자가 문서화 여부 확인 중 다음 요청으로 넘어가서 보류됨**
  - `02_k8s-addon`의 `gavinbunney/kubectl` provider 로컬 캐시 손상(체크섬 불일치) — `.terraform/providers` 삭제 후 재init으로 해결, lock 파일 갱신됨(커밋 필요)
  - Grafana 관리자 비밀번호를 `grafana cli admin reset-admin-password`로 초기화 시도했다가 오히려 admin 계정이 깨짐(Grafana 13.2.0에서 이 CLI가 정상 작동 안 하는 것으로 보임) — Grafana 데이터가 PVC 없이 `emptyDir`(임시 저장소)라는 걸 확인하고 파드를 삭제→재생성해서 Secret 값 기준으로 깨끗하게 복구

---

## [2026-08-19] Claude Code | decision | 대기열 적용 범위 — 예매 플로우만 유지, 홈/상세까지 확장은 보류

이어지는 대화에서 "대기열이 지금 예매할 때만 생기는데 이거 문제 아니냐"는 질문이 나와서 `BookButton.tsx`를 확인 — 대기열(`QueueModal`)이 "예매하기" 클릭 이후에만 뜨고, 홈/로그인/공연상세는 대기열 사각지대임을 코드로 확인. 부하테스트에서 가장 먼저·가장 심하게 무너진 구간(홈 성공률 10%)이 정확히 이 사각지대와 일치한다는 것도 같이 짚음.

- [[decisions/2026-08-19-queue-scope-limited-to-booking-flow]] 신설 — 대기열을 홈까지 확장하는 안(실제 대형 티켓 사이트들의 방식)을 검토했으나, 지금 프로젝트 규모(4인 팀)에서는 과하다고 판단해 **현재 범위 유지로 확정**. 대신 어제 등록한 GitHub 이슈 3건(캐싱, 헬스체크 분리)으로 완화하기로 함. 재검토 트리거(실제 트래픽이 지금 규모를 크게 넘어서면)를 명시적으로 남겨둠
- index.md 갱신

---

## [2026-08-19] Claude Code | troubleshooting | frontend#27(SSR 캐싱) + backend#29(actuator 포트 분리) 코드 반영

같은 날 앞선 세션이 등록해둔 GitHub 이슈 3건 중 frontend#27·backend#29 두 건을 실제로 코드 반영함(frontend#28은 팀원이 별도 진행 중이라 이 세션에서 제외). 두 이슈 모두 "배경/문제/제안"이 이미 구체적으로 작성돼 있어서 그 제안을 그대로 구현.

- **frontend#27** — `Frontend/app/page.tsx`(카테고리/공연목록 fetch 2건), `Frontend/app/events/[performanceId]/page.tsx`(상세 fetch 1건)의 `cache: "no-store"`를 `next: { revalidate: 30 }`로 교체. 30초는 이슈에 적힌 예시값 그대로 채택 — 실제 데이터 변경 주기에 맞춰 팀 논의로 조정 가능.
- **backend#29** — `management.server.port: 8081`을 `application.yml`에 추가하는 것만으로 안 끝나고 4개 레포 동반 수정이 필요했음:
  - `Backend/src/main/resources/application.yml` — `management.server.port: 8081`
  - `Backend/Dockerfile` — `EXPOSE 8081`
  - `Infra/02_k8s-addon/main.tf` — backend Ingress에 `alb.ingress.kubernetes.io/healthcheck-port: "8081"` 추가(target-type=ip라 기본값 traffic-port(8080) 대신 명시적으로 오버라이드해야 함)
  - `CD/helm/templates/backend-deployment.yaml` — containerPort 8081 추가, Service에 `actuator` 포트(8081→8081) 추가
  - `CD/helm/templates/networkpolicy.yaml` + `CD/helm/values.yaml` — `networkPolicy.port`(단일값)를 `networkPolicy.ports`(리스트)로 바꿔서 8081도 allow (ALB의 target-type=ip 헬스체크 트래픽이 이 NetworkPolicy 적용 대상인지 100% 확신은 못 해서, 8080과 동일하게 방어적으로 열어둠 — 실제로 필요 없더라도 해될 것 없음)
- `helm template`(release+prod values 둘 다)과 `terraform validate`(02_k8s-addon)로 렌더링·문법 검증 완료. **네 레포 다 코드 수정만 하고 커밋은 안 함**(frontend/backend/Infra/CD는 팀원이 직접 commit/push하는 레포라 CLAUDE.md 규칙대로 그대로 둠) — 실제 커밋·CI 빌드·ArgoCD sync·배포 후 재검증은 다음 단계.
- [[troubleshooting/loadtest-10000-open-run-cascading-failures]] "2026-08-19 후속" 섹션의 발견 A/B 밑에 반영 내용 추가, 상단에 🟢 배지 추가, 맨 아래 이슈 목록에 상태 갱신
- `wiki/index.md` 갱신 — 72번째 줄(개요 요약)과 105번째 줄(미해결 목록) 상태 갱신, backend#29는 ✅ 해결로 옮기고 frontend#28은 "팀원이 별도 진행 중"으로 남김

**다음에 이어갈 것**: frontend/backend/Infra/CD 4개 레포 각각 커밋+push, CI 빌드로 새 이미지 나오면 `values.yaml`의 image 태그 갱신 → ArgoCD sync → 배포 후 (1) ALB 타겟그룹 헬스체크가 실제로 8081로 붙는지 AWS 콘솔/CLI로 확인 (2) 부하테스트 재실행해서 홈 성공률 개선 여부와 연쇄 장애 재발 여부 검증. frontend#28(헬스체크 전용 엔드포인트)은 팀원 작업 완료되면 같이 재검증.

---

## [2026-08-19] Claude Code | ingest | frontend#27 revalidate 30→60초 조정, backend#29는 팀원이 별도 해결해 원복

frontend#27을 30초로 반영한 직후 사용자가 1분(60초)으로 재조정 요청 — `app/page.tsx`, `app/events/[performanceId]/page.tsx` 값 수정 후 `Frontend` `feature/jwj-frontend-ssr-cache` 브랜치에 커밋 완료(`7cfccbc`). 같은 대화에서 backend#29는 팀원이 이미 별도로 해결했다는 걸 확인 — Backend/Infra/CD에 반영했던 코드는 전부 `git restore`로 원복(커밋 전이라 안전하게 되돌림). [[troubleshooting/loadtest-10000-open-run-cascading-failures]]의 revalidate 값 언급(30→60)과 브랜치/커밋 정보 갱신.

---

## [2026-08-19] Claude Code | troubleshooting | 2000명/4000명 엔드투엔드 부하테스트 성공, 신규 버그 4건 발견·해결

별도 세션에서 AMP/Grafana 트러블슈팅에 이어 진행된 대규모 부하테스트 작업. 로그인→대기열→좌석선택→예매(결제 제외 — 실제 토스페이먼츠 API 호출이라 자동화 불가) 전체 플로우를 k6로 재현하는 `Infra/loadtest/e2e_reservation_2000.js`를 새로 작성하고 반복 실행하며 아래 4가지를 발견·해결함:

- [[troubleshooting/servicemonitor-actuator-port-mismatch]] 신설 — backend#29(actuator 8081 분리)의 사이드이펙트로 Prometheus ServiceMonitor가 옛 포트를 계속 스크랩 중이던 것 발견, `terraform apply -target`으로 해결
- [[troubleshooting/queue-max-active-users-bottleneck]] 신설 — 대기열 `MAX_ACTIVE_USERS=10`이 2000명 규모를 전혀 못 버텨서 테스트 내내 아무도 통과 못하던 것 발견, 150으로 상향(backend hotfix, release 배포 후 dev에도 백포트)
- [[troubleshooting/loadtest-script-response-envelope-gotchas]] 신설 — k6 스크립트가 `GlobalResponseAdvice`의 응답 래핑 규칙(Map은 안 감싸고 DTO/record/List는 `data`로 감쌈)을 몰라서 `queueToken`/`status`가 계속 undefined였던 버그 + 좌석 조회 응답의 `roundId`가 항상 null인 것 발견·우회
- [[troubleshooting/grafana-avg-response-time-rate-nan-artifact]] 신설 — "평균 응답시간" 패널이 버스트성 저트래픽 엔드포인트에서 `rate()` 나눗셈 불안정으로 20013ms 같은 값을 보여주던 문제, 요청률 하한 필터로 해결

위 4건을 전부 고친 뒤 재검증: **2000명 규모 예매 성공률 99.85%(1997/2000), 4000명 규모(좌석 2000석 그대로, 수요=공급의 2배)에서도 정확히 2000명 성공 + 이중예매 0건** — Redis 분산락(`lock:reservation:{seatId}`)이 대규모 동시 경합에서도 완벽하게 작동함을 실측 확인. 4000명 규모에서 KEDA 스케일업 지연(~3분)이 새 병목으로 관찰됨(로그인 실패 343건이 이 구간에 집중) — [[troubleshooting/loadtest-10000-open-run-cascading-failures]] "2026-08-19 추가 후속" 섹션에 정리, 만 명 테스트 때 봤던 완전 다운은 재현 안 됨.

부하테스트용 계정을 `loadtest0001`~`loadtest4000`(4000개)까지 release DB에 시드함(`Infra/loadtest/seed_test_accounts.sql`) — 회원가입/이메일인증 API를 거치지 않고 DB 직접 INSERT.

- `Infra` PR #24 (fix/nyj-servicemonitor-health-port → dev) 머지 완료
- `backend` PR #34 (hotfix/nyj-queue-max-active-users → release) 머지 완료 — 배포까지 확인(ArgoCD `qket-cd` 앱이 자동 sync 안 되는 구조라는 것도 이번에 파악, 수동 SYNCHRONIZE 필요)
- `backend` PR #35 (hotfix/nyj-backport-queue-max-active-users → dev) 머지 완료 — release 반영분 dev 동기화
- index.md, log.md 갱신

**남은 것**: round_id=18의 2000석은 테스트로 전부 RESERVED된 상태로 남아있음(loadtest 계정 소유) — 다음에 이어서 테스트할 사람은 원복 필요(`UPDATE RESERVATIONS SET user_id=NULL, reserved_status='AVAILABLE', reserved_at=NULL WHERE round_id=18 AND user_id LIKE 'loadtest%'`). 결제 플로우 자체의 부하테스트는 여전히 미수행(토스페이먼츠 실제 API 의존 때문에 구조적으로 어려움 — 필요하면 토스 테스트 모드 연동 검토 필요).

---

## [2026-08-20] Claude Code | troubleshooting | 매일 아침 재적용 CRD 순서 문제 3일 연속 재현 — 절차로 정식 반영

어제 index.md에 "미문서화, 다음에 또 겪으면 문서화" 메모를 남겨뒀던 그 문제(`argocd-notifications-eso.tf`의 ESO CRD 역방향 의존)가 오늘 아침 `daily-infrastructure-toggle` 루틴대로 재적용하다 또 재현됨(어제 처음 겪은 ServiceMonitor CRD 문제와 함께, 3일 연속). 이번엔 임시 우회로 넘어가지 않고 제대로 문서화·절차화함.

- [[troubleshooting/crd-not-yet-installed-on-fresh-apply]] 신설 — `kubernetes_manifest`/`kubectl_manifest`가 plan 단계에서 CRD 존재를 요구하는 게 공통 근본 원인임을 정리. 같은 root 내부 순서 문제(ServiceMonitor ← module.monitoring)와 root 경계를 넘는 순서 문제(SecretStore ← 04_data의 module.eso)를 구분해서 설명, 둘 다 "매일 밤 02_k8s-addon을 통째로 destroy하는 비용 절감 루틴" 때문에 최초 1회가 아니라 매일 재현된다는 점 명시
- [[runbook/daily-infrastructure-toggle]] "아침 — 켜기" 절차에 두 targeted apply(`module.monitoring`, `04_data`의 `module.eso`)를 정식 스텝으로 반영 — 다음 사람은 이 문서만 따라가면 안 헤맴
- 근본 리팩터링(ESO 관련 리소스를 `04_data`로 이전)은 아직 미착수로 남김 — 지금은 매일 몇 줄 더 치는 것으로 감수하기로
- index.md 갱신

같은 세션에서 아침 루틴 전체(01_infrastructure → 02_k8s-addon → 04_data release) 실제로 완료함. 별도로, 다른 세션에서 이미 2000/4000명 규모 e2e 부하테스트 성공(성공률 99.85%, 이중예매 0건)까지 진행돼있는 걸 확인 — 만 명 테스트 때의 완전 다운은 더 이상 재현 안 되고, 남은 병목은 KEDA 스케일업 지연(~3분)뿐인 상태.

---

## [2026-08-20] Claude Code | decision + troubleshooting | Ingress → Gateway API 마이그레이션 1단계 완료

Ingress annotation 난립을 계기로 Gateway API 전환을 사용자와 상세히 논의(CRD 개념, kubernetes_manifest의 plan-time 스키마 검증 문제, CD로 빼는 대안의 destroy-순서 리스크 등) 끝에 단계별 마이그레이션으로 합의, 1단계(CRD 설치 + release 환경 파일럿, 실 트래픽 영향 0)를 실제로 구현·검증까지 완료.

- [[decisions/2026-08-20-ingress-to-gateway-api-migration]] 신설 — 왜 CRD+GatewayClass/Gateway/HTTPRoute를 "같은 Helm 차트, 하나의 helm_release"로 설치하는지, CD(ArgoCD)로 빼는 대안을 왜 기각했는지, `LoadBalancerConfiguration.spec.sourceRanges`가 `inbound-cidrs` 갭이 아니었다는 것 등 정리
- [[troubleshooting/alb-controller-gatewayapi-boot-time-crd-check]] 신설 — 실제로 부딪힌 새 버그: AWS Load Balancer Controller가 자기 파드 부팅 시점에 딱 한 번 Gateway API CRD 존재를 확인해서 기능을 켤지 정함(순서가 안 맞으면 `kubectl rollout restart` 수동 필요). `module.gateway_api_crds`(CRD+GatewayClass) → `module.alb_controller` → `module.gateway_api_pilot`(실제 오브젝트) 3단 depends_on으로 근본 해결 — 매일 아침 재적용돼도 재발 안 함
- 실제 apply해서 파이프라인 전체(CRD→GatewayClass→Gateway→ALB 생성→HTTPRoute→TargetGroupConfiguration→ExternalDNS→Route53 레코드) 검증 완료 — `gw-dev.jun979.click` 테스트 호스트네임으로, 기존 `dev.jun979.click`(Ingress 경로) 실트래픽은 전혀 안 건드림
- Terraform 모듈 신설: `Infra/modules/addons/gateway-api-crds`, `Infra/modules/addons/gateway-api-pilot`. `Infra/modules/addons/external-dns`에 `sources`(gateway-httproute 추가) 반영
- 2단계(dev.jun979.click 실제 컷오버)·3단계(admin) 미착수 — 다음 세션에서 이어서 진행

---

## [2026-08-20] Claude Code | troubleshooting | ServiceMonitor/SecretStore도 helm_release로 전환 — CRD 순서 문제 재발 방지 일반화

Gateway API 파일럿에서 검증한 "CRD와 소비 오브젝트를 같은 Helm 차트에 넣고 helm_release로 설치하면 kubernetes_manifest/kubectl_manifest의 plan-time CRD 문제가 원천적으로 없어진다"는 패턴을, 이미 3일 넘게 데었던 기존 두 리소스에도 그대로 적용해서 정리함.

- `kubernetes_manifest.backend_service_monitor` → `modules/addons/backend-servicemonitor`(helm_release)로 전환 — **같은 root(02_k8s-addon) 안** 문제라 완전히 해결됨, 매일 아침 `module.monitoring` targeted apply 불필요해짐
- `kubectl_manifest.argocd_notifications_secret_store`/`_external_secret` → `modules/addons/argocd-notifications-secrets`(helm_release)로 전환 — ESO가 **다른 root(04_data)** 에 있는 cross-root 문제라 완전 해결은 아님. `module.eso` 선적용 런북 절차는 여전히 필요하지만, 순서가 안 맞아도 "이 helm_release 하나만 apply 시점 실패"로 그치고 나머지 무관한 리소스는 정상 적용됨(예전엔 plan 자체가 실패해서 02_k8s-addon 전체가 막혔음)
- 둘 다 실제 apply해서 라이브 클러스터에서 정상 동작 확인(ServiceMonitor 재생성, SecretStore `Valid`/ExternalSecret `SecretSynced`, `argocd-notifications-secret` 값 정상 채워짐)
- [[troubleshooting/crd-not-yet-installed-on-fresh-apply]], [[runbook/daily-infrastructure-toggle]] 갱신 — `module.monitoring` 선적용 스텝 삭제, `module.eso` 선적용은 유지

---

## [2026-08-20] Claude Code | decision | Ingress → Gateway API release+prod 완전 컷오버 완료

1단계(CRD+release 파일럿) 검증 직후, 사용자 판단으로 범위를 확장해서 release(`dev.jun979.click`)+prod(`app.jun979.click`) 둘 다 한 번에 실제 컷오버 — prod가 아직 실서비스 오픈 전이라 단계 분리 없이 진행하기로 함. 기존 `app_ingress_backend`/`app_ingress_frontend`/`faro_ingress` 3개 Ingress 리소스를 코드에서 완전히 삭제.

- 모듈 구조를 4개로 재구성: `gateway-api-crds`(CRD+GatewayClass, 공용)→`alb_controller`→`gateway-api-faro`(alloy-faro ReferenceGrant/TargetGroupConfiguration, release+prod 공유라 한 번만)→`gateway-api-app`(구 gateway-api-pilot, `for_each`로 release/prod 각각 인스턴스화). 공유 리소스를 env별 모듈에 안 두고 따로 뺀 이유: 같은 이름 오브젝트를 두 helm_release가 동시에 소유하려다 충돌하기 때문
- [[troubleshooting/gateway-api-instance-target-type-clusterip-service]] 신설 — `TargetGroupConfiguration.targetType` 기본값(instance)이 ClusterIP 서비스(alloy-faro)에서 Gateway 전체를 실패시키는 문제 발견·해결(`targetType: ip` 명시). 이어서 헬스체크 경로 문제(`/`→404, `/collect`→405)도 Grafana Alloy 표준 엔드포인트 `/-/ready`로 해결
- 실제 apply로 dev.jun979.click(HTTP 200 frontend/backend, HTTP→HTTPS 301 리다이렉트 확인)·app.jun979.click(ALB/DNS까지 정상, backend/frontend는 prod 미배포로 아직 BackendNotFound — 예상된 상태) 검증
- 작업 중 `git reset --hard origin/dev`로 커밋 전 2단계 코드가 한 번 날아갔다가 재작성한 해프닝 있었음(1단계는 이미 커밋되어 있어 안전했고, 라이브 클러스터는 2단계 미적용 상태라 실제 인프라 영향 없었음)
- 3단계(admin — Grafana/ArgoCD)만 미착수로 남음

---

## [2026-08-20] Claude Code | decision | admin(Grafana/ArgoCD) Gateway API 마이그레이션 완료 — dev도 admin 전용으로 재이동, Ingress 완전 소멸

release+prod 컷오버 직후 admin도 이어서 진행. "개발 서버는 관리자만 들어가야 한다"는 판단에 따라 dev.jun979.click(release)를 공개 ALB에서 admin Gateway로 재이동시켜서 grafana/argocd/dev 셋이 팀원 IP 허용목록(`sourceRanges`)을 공유하게 함 — prod는 실서비스용이라 공개 유지.

- `modules/addons/gateway-api-admin` 신설 — Gateway 하나(리스너 3개, 각자 hostname의 HTTPS:443) + http:80 리다이렉트 전용, 인증서 3개를 SNI로 한 리스너에 다중 연결(`defaultCertificate`+`certificates`). `allowedRoutes.namespaces.from: All`로 monitoring/argocd/qket-release에 흩어진 HTTPRoute들의 cross-namespace 첨부 허용
- `admin-ingress.tf`의 `kubernetes_ingress_v1.grafana`/`argocd` 완전 삭제(ACM 인증서 리소스는 재사용), `02_k8s-addon/main.tf`의 `gateway_api_app` for_each에서 release 제거(prod만 남음)
- 실제 컷오버 중 Helm uninstall이 `context deadline exceeded`로 실패하는 걸 겪음 — 실제로는 Gateway/ALB 삭제 자체는 완료됐고 TargetGroupConfiguration 2개만 고아로 남음(수동 정리), Terraform state는 `terraform state rm`으로 직접 정리해서 해결
- 실제 apply + curl로 `dev.jun979.click`/`grafana.jun979.click`/`cd.jun979.click` 전부 검증(타겟그룹 5개 healthy), AWS 보안그룹 직접 조회로 팀원 IP 4개 정상 반영 확인
- **이 시점부터 프로젝트에 Ingress 오브젝트가 완전히 없음** — Ingress → Gateway API 마이그레이션 전체 완료

---

## [2026-08-20] Claude Code | troubleshooting | 2000명 e2e 부하테스트 — DB 풀 사이징/롤아웃 콜드스타트 문제 발견·일부 해결

Gateway API 마이그레이션(release+prod+admin 컷오버) 완료 직후, `Infra/loadtest/e2e_reservation_2000.js`로 실제 부하테스트 진행. 테스트 전 스크립트 자체를 먼저 점검·보강하고, 진행 중 실시간으로 인프라 이상 징후를 여러 건 발견·조치함.

- **k6 스크립트 사전 점검**: `dev.jun979.click`이 그날 admin Gateway로 옮겨지면서 팀원 IP 허용목록에 걸린다는 점 먼저 확인(실행자 IP가 이미 허용목록에 있어 문제없었음). `insecureSkipTLSVerify: true`, `RAMP_SECONDS` 지터(로그인 요청 전)를 `open_run_10000_no_queue.js`와 동일하게 추가 — 기존엔 이 파일에만 빠져있었음
- 부하테스트 도중 backend 파드 재시작 발견 → 조사 결과 KEDA 스케일업+Karpenter 새 노드 콜드스타트 CPU 경합(자가치유, 심각하지 않음)으로 확인
- Grafana 노드 CPU 패널과 AWS 콘솔 노드 목록 수치가 달라 보이는 것에 대한 질문 → CloudWatch 원본 지표로 교차검증, 실제로는 지표 오류가 아니라 부하 자체가 순간적으로 요동치는(bursty) 패턴이라 몇 분 차이로도 수치가 크게 달라짐을 확인
- [[troubleshooting/hikaricp-pool-stale-sizing-after-rds-upgrade]] 신설 — RDS db.t3.micro→medium 업그레이드 이후 `dbPoolSize×maxReplicas` 재계산을 안 해서 실제 한도(341)의 1/3 수준인 120에서 계속 막혀있었던 걸 실측(HikariCP 타임아웃 68건/10분, RDS DatabaseConnections 120 고정)으로 확인·해결(`CD/helm/values.yaml`: dbPoolSize 10→12, maxReplicas 12→20 = 240)
- [[troubleshooting/backend-cold-start-cpu-contention-during-rollout]] 신설 — 위 설정 변경이 트리거한 전체 롤링배포 중, 여러 신규 파드가 한 노드에 몰려서 뜨며 readiness probe가 대거 실패 → `HealthyHostCount`가 18:08 한 순간 1로 붕괴 → 마침 그 순간과 겹친 로그인 요청들이 대량 실패(성공률 57%)로 이어진 걸 CloudWatch로 인과관계까지 확인
- [[troubleshooting/queue-max-active-users-bottleneck]]에 후속 노트 추가 — 로컬에 체크아웃돼있던 다른 브랜치 파일만 보고 "MAX_ACTIVE_USERS가 다시 10으로 돌아갔나?" 착각할 뻔했다가 `origin/release`를 직접 확인해서 150이 맞게 반영돼있음을 재확인(교훈: 로컬 체크아웃 브랜치 ≠ 실제 배포 브랜치). DB 커넥션 여유가 120→240으로 늘어난 만큼 150도 재계산 여지가 있다는 논의는 사용자가 "다음에"로 보류
- CD 레포(`values.yaml`) 변경은 커밋/푸시 안 함 — 사용자가 직접 처리하기로 함(다른 Qket 코드 레포와 동일한 원칙)

---

## [2026-08-21] Claude Code | decision | release DB/Redis를 RDS/ElastiCache → dev-datastore StatefulSet으로 전환

[[decisions/2026-08-21-release-datastore-rds-to-statefulset]] 신설. 비용/관리 부담으로 release는 RDS/ElastiCache를 아예 안 만들기로 하고, 기존에 "앱 동작 확인용"으로만 떠있던 `dev-datastore`(EBS StatefulSet MySQL/Redis)를 release의 유일한 DB/Redis로 승격. 사용자가 release 현재 RDS/ElastiCache 데이터 삭제에 명시 동의(2026-08-21, "응 삭제해도 돼").

- `04_data/main.tf`: `env_config_map`에 `use_managed_datastore`(release=false, prod=true) 추가, `module.rds`/`module.redis`에 `count` 적용 — 이 레포 최초의 환경별 조건부 모듈 생성 패턴. `kubernetes_config_map.app_config`/`module.eso`/`04_data/outputs.tf`는 전부 삼항식(`try()` 포함)으로 분기해서 count=0일 때도 안전하게 fallback
- **`modules/addons/eso`는 코드 한 글자도 안 건드림** — `db-secrets`/`redis-secrets`를 계속 ESO가 관리하되, 호출부에서 넘기는 `rds_master_user_secret_arn`/`rds_endpoint`/`redis_endpoint`만 release일 때 dev-datastore 쪽 값으로 바꿔치기. 신규 `aws_secretsmanager_secret.dev_mysql_root`(03_registry)를 RDS 마스터 시크릿과 동일한 `{username, password}` JSON 모양으로 맞춰서 ESO의 기존 `remoteRef.property` 매핑을 그대로 재사용
- MySQL root 비밀번호를 `02_k8s-addon`(매일 밤 destroy/재생성)의 `random_password`에서 `03_registry`(영구 싱글턴)로 이전 — 기존엔 MySQL 데이터(EBS, 영구보존)와 Terraform이 아는 비밀번호가 둘째 날부터 어긋나는 드리프트 버그가 잠재해있었음(컨테이너가 기존 데이터 디렉터리 있으면 `MYSQL_ROOT_PASSWORD` 재반영 안 함). `ignore_changes`로 보호(`external_api_keys`와 같은 패턴)
- 부수 정리: `dev-datastore` 모듈이 자체 `random_password` 생성을 그만두면서 `02_k8s-addon`에서 `random` provider 제거, `03_registry`에 새로 추가
- `terraform validate`는 3개 root(03_registry/02_k8s-addon/04_data) 전부 통과. count=0 모듈을 삼항식으로 참조해도 안전한지 별도 샌드박스(`var.enabled ? module.thing[0].id : "fallback"`)로 실증 확인. 다만 `01_infrastructure`가 지금(야간) destroy돼있어 `04_data`의 실제 `terraform plan`은 이번 세션에서 못 돌려봄 — **다음 아침 인프라 기동 후 plan/apply로 최종 검증 필요**

---

## [2026-08-21] Claude Code | decision | 04_data를 workspace에서 release/prod 두 디렉토리로 분리

[[decisions/2026-08-21-04data-split-release-prod-directories]] 신설. 위 release datastore 전환으로 `04_data`에 count/삼항식 분기가 늘어난 걸 사용자가 보고, "release는 connection config/시크릿이 이제 완전히 다르니 한 파일로 억지로 합치지 말고 나누자"고 판단 — `04_data`(단일 root+workspace)를 `04_data/release`·`04_data/prod` 완전 독립 두 root로 분리(완전 복사 방식, 공용 모듈로 안 뺌 — 사용자 선택).

- **분리 전 안전 확인**: `aws s3 ls`로 실제 state 파일을 직접 조회해서, prod의 `data` state는 이 시점까지 **단 한 번도 apply된 적이 없었음**을 먼저 확인(release만 `env:/release/data/terraform.tfstate`로 실제 존재) — 이 덕분에 release 쪽만 상태 연속성을 신경 쓰면 되는, 생각보다 안전한 작업이 됨
- `04_data/release`의 backend key를 기존 workspace가 쓰던 것과 **완전히 동일한 문자열**로 맞춰서, `terraform state mv`/`import` 같은 이전 작업 없이 새 디렉토리가 기존 RDS/ElastiCache 등 리소스의 state를 그대로 이어받게 함 — `terraform init` 후 `terraform state list`로 실제로 기존 리소스가 그대로 보이는 것까지 확인. prod는 새 key 사용(이어받을 state가 없었으므로)
- `04_data/release`는 `module.rds`/`module.redis`/그 보안그룹이 아예 없음(count로 숨기는 게 아니라 개념 자체가 없음), `module.eso`는 prod와 동일한 코드를 그대로 재사용하되 입력값만 dev-datastore 쪽으로
- **분리 작업 중 이번 변경과 무관한 잠재 버그 발견**: `modules/addons/eso`의 ESO 컨트롤러(`helm_release "external_secrets"`)가 release/prod 구분 없는 고정 name/namespace라, prod를 처음 apply하면 release가 이미 설치한 것과 충돌할 가능성 — prod가 지금까지 한 번도 apply된 적이 없어서 안 드러났을 뿐, 이 분리와 무관하게 원래 있던 설계 문제. **아직 미해결**, prod를 실제로 켤 때 반드시 먼저 조치 필요
- 작업 도중 이 로컬 체크아웃(`Infra`, 브랜치 `fix/lcy-structural_modification`)에서 `modules/addons/eso/secrets.tf`가 원인 불명으로 변경되어 있던 걸 발견(상세 한국어 주석이 의미 없는 placeholder 헤더로 대체되고, `connection` 시크릿에도 `id: external_api`라고 잘못 붙어있었음) — 사용자가 혼자 작업 중이라고 확인했고, 원본 내용으로 복원함(git checkout이 권한 문제로 막혀서 직접 파일 내용을 다시 씀)
- `Infra/README.md`, `wiki/runbook/terraform-apply-order.md`, `wiki/runbook/daily-infrastructure-toggle.md`, `wiki/architecture/terraform-platform-workload-split.md`(상단에 슈퍼시드 노트 추가)까지 전부 workspace 명령어를 `04_data/release`·`04_data/prod` 경로 기준으로 갱신
- 같은 날 별도로: `modules/addons/eso`의 `kubectl_manifest` yaml_body를 `yamlencode({...})`에서 `manifests/*.yaml.tpl` + `templatefile()`로 리팩터링(사용자 요청 — "매니페스트 관리가 어려우니 yaml 파일로 따로 관리"). 리팩터링 전후 렌더링 결과가 완전히 동일한지 별도 스크래치 테스트로 `yamldecode()` 비교까지 실증 확인(4개 매니페스트 전부 `true`)
- `04_data/release`/`04_data/prod`의 `local.environment`을 하드코딩 문자열 대신 `basename(abspath(path.root))`(폴더 이름 자체를 근거로 삼음)로 변경 — `-chdir` 실행까지 포함해서 별도 스크래치로 실증 확인(`path.cwd`는 `-chdir` 시 틀린 값이 나와서 반드시 `path.root`를 써야 함을 확인)
- [[decisions/2026-08-21-release-datastore-rds-to-statefulset]] 갱신 — `db-secrets`/`redis-secrets`를 ESO 재사용에서 `modules/addons/eso`의 `manage_db_redis_secrets` 스위치 + release 전용 plain `kubernetes_secret`으로 최종 변경(RDS와 달리 dev-mysql은 로테이션이 없어서 ESO의 자동 재동기화가 무의미하다는 게 근거). 그 과정에서 "비밀번호를 매일 밤 재생성하면 안 되는 이유"(MySQL 이미지가 데이터 디렉터리 있으면 MYSQL_ROOT_PASSWORD를 다시 안 읽음), "그럼 실제 로테이션은 어떻게 하나"(ALTER USER 기반 CronJob 설계까지 구체적으로 짚었으나 작업량 대비 실익 낮다고 판단해 기각, 수동 로테이션 절차만 문서화)를 사용자와 논의 — **최종적으로 dev-mysql 비밀번호는 영구 고정(장기 키)으로 의도적 수용, 네트워크가 클러스터 내부 전용이라는 게 근거**. `kubectl exec`/`kubectl port-forward`+Workbench 접근 방법도 함께 정리
- CD 레포 대응(`DB_HOST`/`REDIS_HOST` 하드코딩, `backend.secrets`에서 `redis-secrets` 제거)은 아직 미착수 — 사용자가 직접 처리 예정
