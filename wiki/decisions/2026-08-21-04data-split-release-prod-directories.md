---
title: 04_data를 workspace 대신 release/prod 두 디렉토리로 분리
category: decisions
tags: [terraform, infra, release, prod]
created: 2026-08-21
updated: 2026-08-21
---

---
status: 확정
date: 2026-08-21
author: Claude Code
---

# 04_data를 workspace 대신 release/prod 두 디렉토리로 분리

## 배경

같은 날 먼저 진행한 [[2026-08-21-release-datastore-rds-to-statefulset]]로 release가 RDS/ElastiCache
대신 dev-datastore(StatefulSet)를 쓰게 되면서, 기존 `04_data`(단일 root + `terraform workspace`로
release/prod 분리) 구조에 `count`/삼항식 분기가 여러 곳(`module.rds`/`module.redis`의 count, ESO
입력값 삼항식, config map 값 삼항식, output의 `try()`)에 걸쳐 생겼다. 사용자가 이 상태를 보고 "release는
요소를 아끼기 위해 스테이트풀셋으로 해서 커넥션 콘피그랑 시크릿들이 다 또 틀릴 거다"라고 판단, workspace
전환 실수 방지보다 release/prod 코드 자체를 분리해서 각자 읽기 쉽게 두는 쪽을 택함.

## 결정

- `04_data`(단일 root, workspace)를 `04_data/release`·`04_data/prod` **두 개의 완전히 독립된 root**로
  분리. 서로 다른 S3 backend key를 쓰고, 서로의 존재를 전혀 모른다.
- **완전히 복사(전체 중복) 방식 채택** — `module.storage`/SQS/Lambda/IAM 정책처럼 지금 release/prod가
  완전히 같은 부분까지도 공용 모듈로 안 빼고 각 디렉토리에 그대로 둠. 이유: 공용 모듈로 빼려면 이
  세션에 있는 `04_data` 전체(300줄 가까이)를 감싸는 새 모듈을 만들어야 하는 큰 작업이 되고, 그렇게
  해도 결국 "값만 다르게 넘기는" 지금 매커니즘(env_config_map)과 본질이 같아짐 — 사용자가 명시적으로
  "완전히 복사" 쪽을 선택함.
- `04_data/release`는 `module.rds`/`module.redis`/그 SG(`module.security_group`)가 **아예 없음**
  (count=0으로 숨기는 게 아니라 애초에 그 개념 자체가 없음). `module.eso`는 prod와 동일한 코드를
  그대로 재사용하되, `rds_master_user_secret_arn`/`rds_endpoint`/`redis_endpoint` 입력값만
  `03_registry`의 `dev_mysql_root_secret_arn`/`"dev-mysql"`/`"dev-redis"`로 바꿔치기.
- **release의 S3 backend key는 기존 workspace가 쓰던 것과 완전히 동일한 문자열
  (`env:/release/data/terraform.tfstate`)을 그대로 씀** — 분리 시점에 이미 실제로 살아있던 RDS 등
  리소스의 state가 담긴 키라서, 이 키를 그대로 재사용하면 `terraform state mv`/`import` 같은 이전
  작업 없이 새 디렉토리가 기존 상태를 그대로 이어받는다(실제로 `terraform init` 후 `terraform state
  list`로 기존 리소스가 그대로 보이는 것까지 확인함).
- **prod의 S3 backend key는 새 이름(`prod/data/terraform.tfstate`)** — 분리 시점 기준 이 root가
  단 한 번도 apply된 적이 없었음(S3에 `env:/prod/data/terraform.tfstate` 자체가 없었음, `aws s3 ls`로
  확인) — 이어받을 기존 상태가 없으니 옛 workspace 네이밍을 굳이 유지할 이유가 없었음.

## 고려한 대안

- **기존처럼 단일 root + workspace + count/삼항식 유지** — 처음엔 이 방식으로 구현(`use_managed_datastore`
  bool + `module.rds`/`module.redis`의 `count`)했으나, 사용자가 "connection config/시크릿이 이제 진짜
  다른데 한 파일에 억지로 합쳐두지 말고 나누자"고 판단해서 기각. dev-datastore 도입 이전(release/prod가
  리소스 스펙 값만 다르고 구조는 동일)이었다면 여전히 이 방식이 나았을 것 — [[../architecture/terraform-platform-workload-split]]의 "workspace로 코드 중복을 피한다"는 원칙 자체는 유효하나, 이번엔 그 전제(구조가 같다)가 깨짐.
- **공용 모듈 + release/prod 얇은 wrapper root** — storage/SQS/Lambda 같은 진짜 공통 부분은 모듈로 빼고
  RDS/시크릿처럼 다른 부분만 각 디렉토리에 두는 절충안. 기각(사용자가 "완전히 복사" 선택) — 위 "결정"
  섹션 참고.
- **`04_data-release`/`04_data-prod`처럼 형제 디렉토리로 분리** — 기각, 사용자가 `04_data/release`·
  `04_data/prod`(04_data 자체는 유지하고 그 밑에 하위 디렉토리) 형태를 선택함.

## 트레이드오프 / 남은 리스크

- **코드 중복** — `module.storage`/SQS/Lambda/IAM 정책 등 release/prod가 완전히 같은 부분도 두 파일에
  각각 존재. 앞으로 이 부분을 고칠 일이 생기면 **두 디렉토리를 항상 같이 고쳐야 함** — 하나만 고치고
  깜빡하면 release/prod가 조용히 어긋나는(drift) 위험이 있음. (오늘 이 세션에서 이미 겪은 MySQL
  비밀번호 드리프트, 브랜치 혼동과 같은 성격의 리스크.)
- **⚠️ 이번 분리 작업 중 발견한, 분리와 무관하게 이미 있던 잠재 버그**: `modules/addons/eso`의
  `helm_release "external_secrets"`(ESO 컨트롤러 자체)가 `name`/`namespace`를 environment로 구분하지
  않고 완전히 고정값(`external-secrets`/`external-secrets`)으로 씀 — 원래 "공유 컨트롤러"로 의도된
  것으로 보임(`modules/addons/argocd-notifications-secrets/chart/Chart.yaml` 주석에도 "04_data가 설치한
  ESO(공유 컨트롤러)"라고 되어 있음). 그런데 release/prod가 각자 자기 `module.eso`를 통해 이 리소스를
  **각자의 state로 독립적으로 apply**하므로, prod를 처음 apply하는 순간 release가 이미 설치해둔 것과
  이름이 겹치는 Helm 릴리즈를 또 설치하려다 실패(`cannot re-use a name that is still in use`)할 가능성이
  높음. **이 문제는 오늘 도입한 게 아니라 애초에 workspace 기반 구조에서도 이미 있었음**(prod가 한 번도
  apply된 적이 없어서 지금까지 안 드러났을 뿐). release/prod 분리와 별개로 언젠가 고쳐야 함 — 후보:
  ESO 컨트롤러(helm_release)만 진짜 공유 singleton으로 `02_k8s-addon`(공유 애드온 root)으로 옮기고,
  release/prod 각 `module.eso`는 SecretStore/ExternalSecret(네임스페이스별로 이미 분리돼있음)만 담당하게
  재설계. **아직 미착수.**
- release/prod 각각의 `terraform validate`는 통과했고, release는 `terraform state list`로 기존
  리소스(RDS/Redis/SG 등)를 정상적으로 이어받는 것까지 확인함. 다만 실제 `terraform plan`은 이 분리
  시점 기준 `01_infrastructure`가 야간 destroy 상태라 출력값이 비어있어 확인 못함 — 다음에 인프라가 켜져
  있을 때 `04_data/release`부터 `plan`으로 최종 확인 필요(RDS/ElastiCache/SG가 destroy 목록에 뜨는 게
  맞고, 그 외 예상 밖의 diff가 없는지).
- `03_registry`의 `dev_mysql_root_secret_arn` 출력도 아직 apply 전이라, `04_data/release`가 이 값을
  실제로 읽으려면 `03_registry`를 먼저 apply해야 함 — apply 순서: `03_registry` → (아침 인프라 기동
  후) `04_data/release`.

## 관련
- [[2026-08-21-release-datastore-rds-to-statefulset]]
- [[../architecture/terraform-platform-workload-split]]
- [[../runbook/terraform-apply-order]]
- [[../runbook/daily-infrastructure-toggle]]
