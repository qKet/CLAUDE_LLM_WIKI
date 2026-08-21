---
title: ESO 컨트롤러를 02_k8s-addon의 공유 singleton으로 이전
category: decisions
tags: [terraform, infra, eso, external-secrets, release, prod]
created: 2026-08-21
updated: 2026-08-21
---

---
status: 확정
date: 2026-08-21
author: Claude Code
---

# ESO 컨트롤러를 02_k8s-addon의 공유 singleton으로 이전

## 배경

[[2026-08-21-04data-split-release-prod-directories]]에서 `04_data`를 release/prod 디렉토리로
분리하면서, "분리와 무관하게 이미 있던 잠재 버그"로 다음을 발견/기록해뒀음:

`modules/addons/eso`의 `helm_release "external_secrets"`(ESO 컨트롤러 자체)가 release/prod
구분 없이 고정된 name(`external-secrets`)/namespace(`external-secrets`)를 씀. 그런데
release/prod가 **각자** 자기 `module.eso`를 통해 이 리소스를 각자의 state로 독립적으로
apply하는 구조라, prod를 실제로 처음 apply하는 시점에 두 가지가 순서대로 터짐:

1. **`EntityAlreadyExists`** — `aws_iam_role.eso`가 `team5-qket-eso-role`(고정)이라, release가
   먼저 만든 것과 이름이 겹쳐서 IAM Role 생성이 실패.
2. (1을 environment 접미사로 땜빵한 뒤) **`invalid ownership metadata`** — CRD
   (`external-secrets.io/*`)는 네임스페이스와 무관하게 클러스터에 하나만 존재하고 Helm이
   단일 소유권 메타데이터를 강제하는데, release가 먼저 설치하며 이미 소유권을 가져가서 prod의
   두 번째 `helm_release`가 같은 CRD를 재설치하려다 실패.

당일 실제로 prod를 처음 apply하며 둘 다 재현됨. 급한 대로 `environment` 접미사(역할/네임스페이스
이름) + `install_crds` 변수(먼저 apply되는 쪽만 `true`)로 임시 봉합했으나, 이 방식은:
- ESO 파드가 release/prod 두 벌 뜨는 리소스 낭비
- "어느 쪽이 먼저 apply됐는지"에 따라 `install_crds` 값이 달라지는 순서 의존성 — 재현하기
  까다롭고 팀원이 실수하기 쉬움

둘 다 근본 해결이 아니라서, 사용자가 명시적으로 "공통으로 사용하는 eso가 있으면 02번(k8s-addon)에서
만들면 되는거 아니야?"라고 지적 — 애초에 위 두 문서에서도 후보 해결책으로 이미 언급해뒀던 방향.

## 결정

ESO를 두 부분으로 쪼갠다:

- **컨트롤러(Helm 릴리즈 + CRD + IRSA 역할)** — `modules/addons/eso-controller`(신규)로 빼서
  `02_k8s-addon`에서 **딱 한 번만** 설치. release/prod 구분 없는 진짜 공유 singleton.
- **환경별 동기화 규칙(SecretStore/ExternalSecret) + 그 환경의 시크릿 읽기 IAM 정책** —
  기존 `modules/addons/eso`에 남기되, 더 이상 자기 역할을 만들지 않고 `eso_role_name` 변수로
  받은 공유 역할 이름에 `aws_iam_role_policy`(이름이 서로 다른 인라인 정책)만 추가로 붙임.

`02_k8s-addon`이 `eso_role_name`을 output으로 내보내고, `04_data/release`·`04_data/prod`가
새로 추가한 `data.terraform_remote_state.k8s_addon`으로 그 값을 읽어서 각자 `module.eso`에
넘긴다.

ESO가 서비스어카운트 참조 없이 SecretStore를 만들면(이 프로젝트가 그렇게 씀 — `secret-store.yaml.tpl`에
`auth.jwt.serviceAccountRef` 없음) 컨트롤러 파드 자신의 IRSA 역할로 AWS를 호출하므로, 역할 하나에
release/prod 각자의 정책을 붙이는 것만으로 두 환경 모두 자기 시크릿만 읽을 수 있음(최소 권한 유지 —
서로 다른 인라인 정책이라 release의 정책엔 prod ARN이, prod의 정책엔 release ARN이 없음).

### 부수 효과: cross-root CRD 순서 문제도 같이 해결

`modules/addons/argocd/notifications-secrets`(ArgoCD 알림용 Gmail 자격증명 동기화)가 겪던
별개의 문제 — ESO가 `04_data`(02_k8s-addon보다 **나중**에 apply되는 root)에 있어서, 매일 아침
"04_data의 module.eso를 먼저 apply해야 하는" 런북 절차가 필요했던 것([[crd-not-yet-installed-on-fresh-apply]]
참고) — 도 이번 이전으로 자연스럽게 사라짐. ESO 컨트롤러가 이제 `02_k8s-addon`(notifications-secrets와
같은 root)에 있으므로 `module.argocd`가 `depends_on=[module.eso_controller]`로 같은 root 안에서
순서를 보장받는다. 확립된 root apply 순서(infrastructure→k8s-addon→data) 자체가 "컨트롤러가 항상
그 소비자보다 먼저 존재"를 만족시키는 방향이 됐기 때문 — 이전엔 반대(소비자가 컨트롤러보다 먼저인
root에 있었음)라 문제였던 것.

## 고려한 대안

- **환경 접미사 + `install_crds` 토글 유지** — 기각. 순서 의존성이 있고(먼저 apply된 쪽만
  `install_crds=true`), 컨트롤러가 두 벌 뜨는 낭비가 계속됨. 애초에 "공유 singleton"으로
  의도된 리소스를 억지로 환경별로 쪼갠 것이었으므로 근본 해결이 아님.
- **CRD만 별도 root(예: 03_registry)에 두고 컨트롤러는 그대로 각 환경에** — 기각. 검토는 안 함,
  컨트롤러 자체(파드/역할)가 환경 구분이 필요 없는 리소스라 컨트롤러 전체를 옮기는 게 더 단순함.
- **04_data 자체를 병합해서 ESO를 한 곳에서만 호출** — 기각. [[2026-08-21-04data-split-release-prod-directories]]에서
  이미 release/prod가 모듈 유무 자체가 다른 구조라 분리하기로 확정했고, 이번 문제는 그 분리와
  무관하게 "ESO 컨트롤러가 있어야 할 자리(공유 애드온 root)에 있지 않았던" 게 원인이라 04_data
  병합은 다른 문제를 잘못된 해법으로 푸는 것.

## 트레이드오프 / 남은 리스크

- `02_k8s-addon`이 매일 밤 destroy→아침 재생성되는 root이므로, ESO 컨트롤러도 그 주기를 그대로
  탄다 — 이미 `module.dev_datastore`/`module.argocd` 등 다른 애드온과 같은 처지라 새로운 리스크는
  아니지만, "아침에 02_k8s-addon부터 apply해야 04_data의 ExternalSecret이 정상 동작한다"는 순서
  제약이 이제 ESO에도 명시적으로 적용된다는 점은 기록해둠(기존 `daily-infrastructure-toggle` 런북의
  일반 순서와 동일해서 별도 절차 추가는 없음).
- `modules/addons/eso`의 `helm`/`kubectl` provider 요구사항 중 `helm`은 이제 이 모듈이 안 써서
  `04_data/release`·`04_data/prod`의 `providers.tf`에서 제거함 — 혹시 이 root에 나중에 다른
  `helm_release`가 추가되면 그때 다시 넣어야 함.
- `terraform validate`는 `02_k8s-addon`/`04_data/release`/`04_data/prod` 전부 통과 확인함.
  다만 release는 이미 environment-접미사 이름(`team5-qket-eso-role-release` 등)으로 실제
  IAM 역할/Helm 릴리즈가 살아있는 상태 — 이 설계를 실제로 `apply`할 때는 새 공유 역할
  (`team5-qket-eso-role`)이 새로 생성되고 예전 접미사 붙은 리소스(`team5-qket-eso-role-release`,
  `external-secrets-release` 네임스페이스의 Helm 릴리즈)는 고아로 남으므로, apply 전에
  **수동으로 정리(예: `helm uninstall`, `terraform state rm` 또는 그냥 남겨두고 나중에 별도
  destroy)가 필요함 — 아직 미착수.** prod는 한 번도 정상적으로 안착한 적이 없어서(이름 충돌로
  실패했었음) 이 문제가 없음.

## 관련

- [[2026-08-21-04data-split-release-prod-directories]] — 이 버그를 처음 발견/기록한 문서
- [[2026-08-21-release-datastore-rds-to-statefulset]]
