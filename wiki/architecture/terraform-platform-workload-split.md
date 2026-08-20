---
title: infrastructure / data 2-root 구조와 workspace (구 platform/workload)
category: architecture
tags: [infra, terraform]
created: 2026-08-06
updated: 2026-08-11
---

# infrastructure / data 2-root 구조와 workspace

> ⚠️ 2026-08-10에 root 디렉토리 이름이 바뀌었다: `platform` → `infrastructure`, `workload` → `data` (기능은 동일, 이름만 변경). 이 파일명(`terraform-platform-workload-split`)은 옛 이름 그대로 두되(다른 문서들의 링크가 이 파일명을 가리키고 있어서), 본문 내용은 새 이름 기준으로 갱신했다. 과거 로그(`log.md`)나 ADR에서 `platform`/`workload`라고 쓴 건 그 시점 실제 이름이므로 안 고친다.
>
> ⚠️ **2026-08-21 갱신: `data`는 더 이상 workspace로 안 나뉜다.** release가 RDS/ElastiCache 대신 dev-datastore(StatefulSet)를 쓰게 되면서 release/prod 구조 자체가 달라져, 아래 "`data` 안에서 release/prod를 나누는 법: terraform workspace" 절은 **`04_data/release`·`04_data/prod` 두 개의 독립된 root로 대체됨** — [[../decisions/2026-08-21-04data-split-release-prod-directories]] 참고. 이 문서의 나머지(왜 `infrastructure`/`data`를 root 자체로 나눴는지, SG를 CIDR로 참조하는 이유, `try()` 패턴)는 여전히 유효.

## 구조

`Infra`에는 root가 두 개다:

- **`infrastructure/`**(구 `platform`) — VPC/서브넷/EKS/bastion/ECR. **공유 싱글턴** — release/prod가 나눠 갖지 않고 딱 하나만 존재. workspace 안 씀, 그냥 한 번 apply. 매일 수동으로 켜고 끄는 대상.
- **`data/`**(구 `workload`) — RDS/Redis/S3(포스터)/CloudFront/네임스페이스/app-config. **release/prod마다 따로** 존재해야 함 — `terraform workspace`로 분리. 실제 데이터를 담고 있어 **절대 지우지 않는** 대상.

`data`는 `infrastructure`가 만든 값(vpc_id, subnet ids, eks 클러스터 정보 등)을 `terraform_remote_state`로 읽어서 쓴다 — 자세한 매커니즘은 [[terraform-remote-state]].

## 왜 나눴나 (workspace 하나로 다 못 하는 이유)

만약 `infrastructure`까지 포함해서 전부 workspace로만 나누면, `terraform workspace select release && apply` / `terraform workspace select prod && apply`를 각각 돌릴 때마다 **VPC도 EKS도 워크스페이스별로 따로 생긴다** — release용 EKS, prod용 EKS 두 개가 되어버려서 EKS 컨트롤플레인 비용이 거의 2배가 되고, 애초에 "VPC/EKS는 공유"라는 요구사항에 안 맞는다. 그래서 공유해야 하는 것(`infrastructure`)과 환경별로 달라야 하는 것(`data`)을 root 자체를 나누는 걸로 분리하고, `data` 안에서만 workspace를 쓴다.

## `data` 안에서 release/prod를 나누는 법: terraform workspace

```bash
cd Infra/04_data
terraform workspace new release   # 최초 1회
terraform workspace new prod      # 최초 1회
terraform workspace select release
terraform apply
```

S3 backend + workspace 조합이면 state key가 자동으로 `env:/<workspace>/data/terraform.tfstate`로 나뉘어서, release/prod가 물리적으로 완전히 다른 state 파일에 저장된다.

## `default` workspace 실수 방지: `workspace_guard`

`terraform init` 직후엔 항상 `default`라는 workspace에 있다. 여기서 실수로 `apply`하면 안 되므로 `04_data/main.tf`에 이런 가드를 넣어뒀다:

```hcl
resource "terraform_data" "workspace_guard" {
  lifecycle {
    precondition {
      condition     = contains(["release", "prod"], terraform.workspace)
      error_message = "workspace는 'release' 또는 'prod'여야 합니다..."
    }
  }
}
```

`terraform.workspace`를 직접 조회하는 `env_config`(아래)가 `default`에서도 `terraform validate`/`plan`이 깨지지 않도록 `lookup()`으로 release 기본값에 폴백해두고, **실제 apply 차단은 이 precondition이 전담**한다 (validate 단계에서 미리 죽으면 이 친절한 에러 메시지 대신 밋밋한 "Invalid index"만 보임).

## release/prod마다 달라지는 값: `env_config` locals 맵

`db_instance_class`, `multi_az`, `skip_final_snapshot`, `deletion_protection`, `redis_node_type`, `force_destroy` 같은 값들은 release/prod가 달라야 한다. `04_data/main.tf`의 locals에 workspace-키 맵으로 정의:

```hcl
locals {
  environment = terraform.workspace
  env_config_map = {
    release = { db_instance_class = "db.t3.micro", multi_az = false, skip_final_snapshot = true, ... }
    prod    = { db_instance_class = "db.t3.small", multi_az = true,  skip_final_snapshot = false, ... }
  }
  env_config = lookup(local.env_config_map, local.environment, local.env_config_map["release"])
}
```

> 이전엔(재편 전) "prod를 실제로 켤 때는 코드를 직접 고쳐서 안전장치(`skip_final_snapshot=false` 등)를 켤 것"이라는 주석에 의존했었다. 이제는 `terraform workspace select prod`만 하면 이 값들이 **자동으로** 안전한 쪽으로 바뀐다 — 수동으로 빠뜨릴 여지를 없앤 개선.

## `data`가 `infrastructure`의 SG를 참조하면 안 되는 이유 (2026-08-10)

`infrastructure`는 VPC/EKS/bastion처럼 자주 destroy/재생성하는 대상이고, `data`(RDS/Redis)는 실제 데이터를 담고 있어 **절대 같이 지워지면 안 되는** 대상이다 — 이 둘의 destroy 주기가 근본적으로 다르다는 게 이 구조의 핵심 전제다.

처음엔 `data`의 rds/redis 보안그룹 ingress 규칙이 `infrastructure`의 bastion/EKS SG를 **SG ID로 직접 참조**하고 있었다(`security_groups = [data.terraform_remote_state.infrastructure.outputs.bastion_security_group_id]`). 이러면:

1. `infrastructure`를 destroy하려 하면 AWS가 `DependencyViolation`으로 거부한다 — `data`의 SG 규칙이 아직 그 SG를 참조 중이라서. `data`는 절대 안 지우는 게 원칙인데, 그럼 `infrastructure`도 영원히 못 지운다는 모순이 생김.
2. 설령 순서를 억지로 맞춰서 지운다 해도, `infrastructure`를 재생성하면 bastion/EKS SG는 **새 ID**로 다시 생기므로 `data`가 참조하던 옛 SG ID는 죽은 참조가 되고, `data`도 다시 apply해야 함 — "infrastructure만 따로 껐다 켰다" 한다는 목표에 어긋남.

**해결**: SG ID 참조 대신 **CIDR 대역**으로 ingress를 검. bastion과 EKS 노드/파드가 모두 `private_general_subnet`에 있으므로, `infrastructure`가 이 서브넷의 CIDR 목록(`private_general_subnet_cidrs`, 서브넷 CIDR은 변수라 재생성해도 안 바뀜)을 output으로 내보내고 `data`는 그 CIDR로만 ingress를 허용한다.

```hcl
# 04_data/main.tf — rds/redis SG
ingress = [{
  cidr_blocks     = data.terraform_remote_state.infrastructure.outputs.private_general_subnet_cidrs
  security_groups = []
  ...
}]
```

트레이드오프: "정확히 이 SG(=이 ENI들)에서 오는 트래픽만"에서 "이 서브넷 대역에서 오는 트래픽"으로 범위가 살짝 넓어짐 — 그래도 여전히 VPC 프라이빗 서브넷 내부로 한정되므로 인터넷 노출 등의 리스크는 없음. 대신 `infrastructure`↔`data` 사이의 AWS 레벨 하드 의존이 완전히 사라져서, `infrastructure`를 몇 번을 destroy/재생성해도 `data`가 전혀 영향받지 않게 됨.

> ✅ 해결됨(2026-08-10): `helm_release.argocd`/`kubernetes_namespace.qket`(kubernetes/helm provider 사용)가 이 CIDR 변경과는 별개로, `infrastructure`를 destroy할 때 그 provider가 인증하는 데 쓰는 EKS Access Entry가 먼저 지워지면서 `Unauthorized`가 나는 문제가 있었다 — [[eks-provider-auth]], [[../troubleshooting/eks-destroy-layer-separation]] 참고. 이 두 리소스를 `infrastructure`에서 분리해 별도 root `02_k8s-addon`으로 옮겨서 해결함.

## `try()`로 nightly-off 상태의 `-refresh-only`/`plan`을 방어하는 패턴

`01_infrastructure`는 매일 밤 destroy되는 대상([[../runbook/daily-infrastructure-toggle]])이라, 아침에 다시 apply되기 전까지는 `bastion` 보안그룹처럼 "밤엔 존재하지 않는 리소스"가 실제로 있다. 문제는 Terraform이 `-refresh-only`/`plan`을 돌릴 때 **그 리소스가 지금 state에 있든 없든 config 표현식 자체는 항상 평가한다**는 것 — `module.security_group.security_group_ids["bastion"]`처럼 map 인덱싱을 직접 쓰면, bastion SG가 없는 밤 시간대엔 `Invalid index` 에러로 `-refresh-only`가 통째로 하드-실패한다(이 값을 실제로 쓰는 리소스가 지금 존재하냐 마냐와 무관하게, config에 그 표현식이 적혀있다는 사실만으로 에러가 남).

`01_infrastructure/outputs.tf`의 `bastion_security_group_id` output과 `main.tf`의 `module "ec2"`가 받는 `security_group_id` 인자, 이 두 곳 모두 실제로 이 에러를 겪었고, `try(module.security_group.security_group_ids["bastion"], null)`로 감싸서 해결했다.

```hcl
# 밤에 bastion SG가 없어도 -refresh-only/plan이 하드-에러 없이 null로 넘어가게
output "bastion_security_group_id" {
  value = try(module.security_group.security_group_ids["bastion"], null)
}
```

이 패턴은 "밤엔 없을 수 있는 리소스를 참조하는 모든 표현식"에 재사용 가능한 일반 해법 — 앞으로 nightly-toggle 대상에 optional 리소스를 추가할 때마다 같은 이유로 필요해질 수 있다.

## 관련
- [[terraform-module-boundaries]]
- [[terraform-remote-state]]
- [[2026-08-06-terraform-module-restructure]]
- [[../troubleshooting/eks-provider-auth]]
- [[../troubleshooting/eks-destroy-layer-separation]]
- [[../troubleshooting/destroy-order-incident-and-webhook-orphans]]
- [[../runbook/daily-infrastructure-toggle]]
