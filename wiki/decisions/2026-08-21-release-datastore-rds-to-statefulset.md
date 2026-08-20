---
title: release 환경 DB/Redis를 RDS/ElastiCache에서 dev-datastore StatefulSet으로 전환
category: decisions
tags: [terraform, rds, redis, cost, release, eso]
created: 2026-08-21
updated: 2026-08-21
---

---
status: 확정
date: 2026-08-21
author: Claude Code
---

# release 환경 DB/Redis를 RDS/ElastiCache에서 dev-datastore StatefulSet으로 전환

## 배경

`release`/`prod` 둘 다 진짜 RDS(db.t3.medium)/ElastiCache(cache.t3.micro)를 띄우고 있었는데, 비용과
관리 부담(야간 destroy/재생성 사이클에 매번 딸려오는 RDS 생성 시간, 스냅샷/암호화 이슈 등 —
[[../troubleshooting/hikaricp-pool-stale-sizing-after-rds-upgrade]] 같은 RDS 자체 이슈도 계속 나옴)이
있다는 팀 판단. `release`는 이미 `02_k8s-addon`에 `modules/addons/dev-datastore`(EBS 기반 StatefulSet
MySQL/Redis, "앱 동작 확인용")가 떠 있었으므로, 이 파드를 release의 **유일한** DB/Redis로 승격하고
RDS/ElastiCache를 아예 만들지 않기로 함. `prod`는 그대로 진짜 RDS/ElastiCache를 유지.

## 결정

- `04_data`의 `env_config_map`에 `use_managed_datastore` 필드 추가 (`release = false`, `prod = true`)
- `module "rds"`/`module "redis"`(둘 다 `04_data/main.tf`)에 `count = local.env_config.use_managed_datastore ? 1 : 0` 추가 — 이 레포에 환경별 조건부 모듈 생성 전례가 없어서 이번에 새로 도입한 패턴. 참조하는 쪽(`kubernetes_config_map.app_config`, `module.eso`, `04_data/outputs.tf`)은 전부 `local.env_config.use_managed_datastore ? module.rds[0].xxx : <dev-datastore 대체값>` 삼항식(또는 `try()`)으로 분기
- **`module.eso`(External Secrets Operator) 자체는 한 글자도 안 건드림.** `db-secrets`/`redis-secrets`를 계속 ESO가 관리하되, 호출부(`04_data`)에서 넘기는 입력값만 release일 때 dev-datastore 쪽으로 바꿔치기:
  - `rds_master_user_secret_arn` → release는 `03_registry`에 새로 만든 `aws_secretsmanager_secret.dev_mysql_root`(모양을 RDS 마스터 시크릿과 동일하게 `{username, password}` JSON으로 맞춤 — 그래서 ESO의 `remoteRef.property = "username"/"password"` 매핑이 그대로 통함)
  - `rds_endpoint`/`redis_endpoint`(connection 시크릿의 DB_HOST/REDIS_HOST 값으로만 쓰임) → release는 실제 엔드포인트 대신 같은 네임스페이스 K8s 서비스명 `"dev-mysql"`/`"dev-redis"`
- **MySQL root 비밀번호를 `03_registry`(불변 싱글턴 레이어)로 이전.** 기존엔 `dev-datastore` 모듈 안에서 `random_password.mysql_root`로 직접 생성했는데, 이 모듈이 속한 `02_k8s-addon`은 매일 밤 통째로 destroy/재생성되는 반면 MySQL 데이터(EBS 볼륨)는 영구 보존이라, 컨테이너가 재생성 둘째 날부터는 최초 부트스트랩 때만 읽는 `MYSQL_ROOT_PASSWORD` 환경변수를 다시 반영하지 않음 — Terraform이 알고 있는 값과 실제 MySQL 안 비밀번호가 어긋나는 드리프트 버그가 잠재해 있었음(EBS 볼륨을 `03_registry`로 옮긴 것과 정확히 같은 이유). `random_password.dev_mysql_root` + `aws_secretsmanager_secret.dev_mysql_root`를 `ignore_changes = [secret_string]`로 보호(`external_api_keys`와 같은 패턴)해서 03_registry에 두고, `dev-datastore` 모듈은 이제 `mysql_root_password` 변수로 받기만 함. `02_k8s-addon`은 `data "aws_secretsmanager_secret_version"`으로 이 값을 직접 읽어서 모듈에 전달(K8s ESO를 거치지 않음 — Terraform 실행 주체가 이미 이 시크릿을 읽을 IAM 권한을 갖고 있어서 별도 IRSA 배선 불필요)
- 부수 정리: `dev-datastore` 모듈이 더 이상 `random` provider를 안 쓰게 돼서 `02_k8s-addon/providers.tf`에서 해당 provider 블록 제거, 대신 `03_registry/providers.tf`에 새로 추가

## 고려한 대안

- **ESO에 release 전용 분기(`manage_db_redis_secrets` 변수 등)를 넣어 `db-secrets`/`redis-secrets`를 plain `kubernetes_secret`으로 따로 만드는 방식** — 기각. RDS 마스터 시크릿과 "같은 JSON 모양"으로 맞추기만 하면 eso 모듈이 이미 하는 일(SecretStore→ExternalSecret→K8s Secret 동기화)을 100% 재사용할 수 있는데, 굳이 새 변수/새 리소스를 추가해서 같은 이름(`db-secrets`) 시크릿을 두 가지 다른 메커니즘이 만들려고 경합하게 만들 이유가 없음
- **02_k8s-addon에서 remote-state로 04_data 출력을 읽어 plain Secret 생성** — 기각. `04_data`가 `02_k8s-addon`보다 나중에 apply되는 순서라(namespace가 k8s-addon 소관) 반대 방향 참조는 순환 의존이 됨 — 기존 db-secrets/redis-secrets가 애초에 04_data에 있는 이유와 동일

## 트레이드오프 / 남은 리스크

- dev-mysql/dev-redis는 단일 파드(StatefulSet replicas=1)라 RDS Multi-AZ/ElastiCache 이중화가 전혀 없음 — release가 "저렴/자주재생성 샌드박스"라는 전제를 그대로 이어받는 것이므로 의도된 트레이드오프. **prod에는 이 변경이 전혀 적용되지 않음.**
- 이 apply를 실행하면 **현재 release에 떠 있는 실제 RDS 인스턴스/ElastiCache 클러스터가 삭제됨** — 사용자가 명시적으로 삭제 동의함(2026-08-21, "응 삭제해도 돼").
- `terraform plan`으로 실제 상태 대비 최종 검증은 아직 안 함 — 세 root(`03_registry`/`02_k8s-addon`/`04_data`) 모두 `terraform validate` 통과, 그리고 `count`+삼항식으로 존재하지 않는 모듈 인스턴스를 참조해도 안전한지는 별도 샌드박스에서 실증 확인함(`var.enabled ? module.thing[0].id : "fallback"` 패턴, count=0일 때 정상적으로 fallback 값 반환). 다만 `01_infrastructure` state가 지금(야간) 비어있어 `04_data`의 실제 `terraform plan`은 이번 세션에서 못 돌려봄 — 다음 아침 인프라 기동 후 `terraform plan`/`apply`로 최종 확인 필요.

## 관련
- [[../troubleshooting/hikaricp-pool-stale-sizing-after-rds-upgrade]]
- [[../runbook/daily-infrastructure-toggle]]
- [[../architecture/terraform-platform-workload-split]]
