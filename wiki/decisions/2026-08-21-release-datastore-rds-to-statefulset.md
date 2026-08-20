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

- `04_data`의 `env_config_map`에 `use_managed_datastore` 필드 추가 (`release = false`, `prod = true`) — 이후 [[2026-08-21-04data-split-release-prod-directories]]로 `04_data` 자체가 release/prod 디렉토리로 분리되면서, 이 필드는 사라지고 `04_data/release`는 애초에 `module.rds`/`module.redis`가 없는 형태로 정리됨(아래 "2026-08-21 갱신" 참고).
- **MySQL root 비밀번호를 `03_registry`(불변 싱글턴 레이어)로 이전.** 기존엔 `dev-datastore` 모듈 안에서 `random_password.mysql_root`로 직접 생성했는데, 이 모듈이 속한 `02_k8s-addon`은 매일 밤 통째로 destroy/재생성되는 반면 MySQL 데이터(EBS 볼륨)는 영구 보존이라, 컨테이너가 재생성 둘째 날부터는 최초 부트스트랩 때만 읽는 `MYSQL_ROOT_PASSWORD` 환경변수를 다시 반영하지 않음 — Terraform이 알고 있는 값과 실제 MySQL 안 비밀번호가 어긋나는 드리프트 버그가 잠재해 있었음(EBS 볼륨을 `03_registry`로 옮긴 것과 정확히 같은 이유). `random_password.dev_mysql_root` + `aws_secretsmanager_secret.dev_mysql_root`를 `ignore_changes = [secret_string]`로 보호(`external_api_keys`와 같은 패턴)해서 03_registry에 두고, `dev-datastore` 모듈은 이제 `mysql_root_password` 변수로 받기만 함. `02_k8s-addon`은 `data "aws_secretsmanager_secret_version"`으로 이 값을 직접 읽어서 모듈에 전달(K8s ESO를 거치지 않음 — Terraform 실행 주체가 이미 이 시크릿을 읽을 IAM 권한을 갖고 있어서 별도 IRSA 배선 불필요)
- 부수 정리: `dev-datastore` 모듈이 더 이상 `random` provider를 안 쓰게 돼서 `02_k8s-addon/providers.tf`에서 해당 provider 블록 제거, 대신 `03_registry/providers.tf`에 새로 추가

### 2026-08-21 갱신 — db-secrets/redis-secrets를 ESO 대신 직접 관리하기로 최종 변경

처음엔 `module.eso`(ESO)를 그대로 재사용해서(ARN만 dev_mysql_root로 바꿔치기) `db-secrets`/`redis-secrets`를 만들었는데, 다시 검토하면서 RDS와 dev-mysql의 근본적 차이(RDS는 자동 로테이션이 있어 ESO의 주기적 재동기화가 실제로 의미 있지만, dev-mysql은 `ignore_changes`로 보호돼 있어 애초에 절대 안 바뀜)를 짚고 아래처럼 바꿈:

- `modules/addons/eso`에 `manage_db_redis_secrets`(기본 true) 변수 추가 — false면 `connection` 시크릿과 `external_secret_db`/`external_secret_redis`를 아예 안 만듦(external-api-secrets는 이 플래그와 무관하게 계속 관리 — 콘솔에서 수동 로테이션한 값을 ESO가 자동으로 반영해주는 실사용 워크플로우가 있어서).
- `04_data/release`는 `manage_db_redis_secrets = false`로 호출하고, `db-secrets`는 `data "aws_secretsmanager_secret_version"`(02_k8s-addon의 dev-mysql 컨테이너 시크릿과 같은 패턴) + plain `kubernetes_secret`으로 직접 생성. **DB_HOST 키는 이 Secret에 안 넣음.**
- **DB_HOST/REDIS_HOST는 `kubernetes_config_map.app_config`(`DB_PORT`/`DB_NAME`/`REDIS_PORT`가 이미 있던 그 ConfigMap)에 `"dev-mysql"`/`"dev-redis"` 고정값으로 그대로 추가**(2026-08-21 정정 — 처음엔 "CD에서 하드코딩" 방향으로 얘기했다가, `app-config`가 이미 CD의 `configMaps` 목록에 들어있어서 **CD 코드를 전혀 안 건드리고** Terraform 쪽에서만 끝낼 수 있다는 걸 확인하고 변경).
- **`redis-secrets`는 release에서 아예 안 만듦** — dev-redis는 애초에 인증(비밀번호) 자체가 없어서 이 Secret엔 원래 REDIS_HOST 하나만 있었는데, 그마저 ConfigMap으로 옮기면 담을 값이 없어짐. **CD 쪽에서 `backend.secrets` 목록에서 `redis-secrets`만 빼면 됨(미착수, 사용자가 직접 처리 예정)** — 안 하면 release 배포 시 파드가 존재하지 않는 Secret을 참조해서 못 뜰 위험 있음.

## 고려한 대안

- **ESO에 release 전용 분기 없이 그대로 재사용(ARN만 바꿔치기)** — 처음엔 이렇게 했다가 위 이유로 변경. `04_data/prod`는 여전히 이 방식(ESO 그대로) 그대로 씀.
- **02_k8s-addon에서 remote-state로 04_data 출력을 읽어 plain Secret 생성** — 기각. `04_data`가 `02_k8s-addon`보다 나중에 apply되는 순서라(namespace가 k8s-addon 소관) 반대 방향 참조는 순환 의존이 됨.
- **DB 비밀번호를 실제로 로테이션하는 CronJob(ALTER USER + Secrets Manager/K8s Secret 동시 업데이트)** — 논의만 하고 기각(2026-08-21). 새 컨테이너 이미지(mysql client+aws cli+kubectl)/IRSA/RBAC/CronJob까지 다 새로 만들어야 하는 작업량인데, dev-mysql이 실사용자 데이터가 아니고 네트워크도 클러스터 내부로만 열려있어서(아래 참고) 투자 대비 이득이 낮다고 판단. **필요해지면 나중에 재검토하기로 함.**
- **수동 로테이션**(사람이 직접 `ALTER USER`로 실제 DB 비밀번호를 바꾸고 Secrets Manager 값도 손으로 맞춘 뒤 `04_data/release` 재적용) — 새 인프라 없이 가능한 대안으로 확인만 해두고 지금 당장 실행하지는 않음. `ignore_changes`가 이미 이 수동 갱신을 막지 않게 돼있음(`external_api_keys`와 동일 패턴).

## 트레이드오프 / 남은 리스크

- dev-mysql/dev-redis는 단일 파드(StatefulSet replicas=1)라 RDS Multi-AZ/ElastiCache 이중화가 전혀 없음 — release가 "저렴/자주재생성 샌드박스"라는 전제를 그대로 이어받는 것이므로 의도된 트레이드오프. **prod에는 이 변경이 전혀 적용되지 않음.**
- 이 apply를 실행하면 **현재 release에 떠 있는 실제 RDS 인스턴스/ElastiCache 클러스터가 삭제됨** — 사용자가 명시적으로 삭제 동의함(2026-08-21, "응 삭제해도 돼").
- **dev-mysql root 비밀번호는 사실상 영구 고정(장기 키)** — 데이터(EBS)가 영구 보존인 이상, 컨테이너 재부팅 시 env var를 다시 안 읽는 MySQL 이미지 특성 때문에 실제 로테이션이 불가능함(로테이션하려면 위 "고려한 대안"의 CronJob/수동 방식처럼 살아있는 DB에 직접 `ALTER USER`를 실행해야 함). **의도적으로 수용한 리스크** — dev-mysql/dev-redis는 headless Service(`cluster_ip: None`)라 LoadBalancer/NodePort 없이 클러스터 내부에서만 닿을 수 있고 인터넷 노출이 전혀 없어서, 이 비밀번호가 유출될 경로 자체가 제한적이라고 판단(2026-08-21). 사람이 직접 조회/접속하려면 `kubectl exec`(CLI) 또는 `kubectl port-forward` + MySQL Workbench(GUI) — 둘 다 네트워크 노출과 무관하게 kubectl/EKS 권한만으로 동작.
- 비밀번호가 이제 Terraform state 3곳(`03_registry`의 `random_password` 원본, `02_k8s-addon`의 dev-mysql 컨테이너 시크릿, `04_data/release`의 `db-secrets`)에 평문으로 존재 — RDS(AWS가 자동 관리, Terraform state에 전혀 안 남음)와 달리 완전 무노출은 애초에 불가능(`random_password`를 쓰는 이상 원본 상태 파일엔 항상 남음). ESO를 계속 썼다면 `04_data` 쪽 한 곳은 줄일 수 있었지만, RDS와 달리 로테이션 이점이 없어서 그 정도 축소를 위해 ESO를 유지할 실익은 낮다고 판단하고 받아들임.
- `terraform plan`으로 실제 상태 대비 최종 검증은 아직 안 함 — 다음 아침 인프라 기동 후 `03_registry` → `04_data/release` 순서로 apply/plan 확인 필요.

## 관련
- [[2026-08-21-04data-split-release-prod-directories]]
- [[../troubleshooting/hikaricp-pool-stale-sizing-after-rds-upgrade]]
- [[../runbook/daily-infrastructure-toggle]]
- [[../architecture/terraform-platform-workload-split]]
