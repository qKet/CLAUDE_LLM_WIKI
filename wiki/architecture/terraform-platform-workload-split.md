---
title: platform / workload 2-root 구조와 workspace
category: architecture
tags: [infra, terraform]
created: 2026-08-06
updated: 2026-08-06
---

# platform / workload 2-root 구조와 workspace

## 구조

`Infra/terraform`에는 root가 두 개다:

- **`platform/`** — VPC/서브넷/EKS/bastion/ECR. **공유 싱글턴** — release/prod가 나눠 갖지 않고 딱 하나만 존재. workspace 안 씀, 그냥 한 번 apply.
- **`workload/`** — RDS/Redis/S3(포스터)/CloudFront/네임스페이스/app-config. **release/prod마다 따로** 존재해야 함 — `terraform workspace`로 분리.

`workload`는 `platform`이 만든 값(vpc_id, subnet ids, eks 클러스터 정보 등)을 `terraform_remote_state`로 읽어서 쓴다 — 자세한 매커니즘은 [[terraform-remote-state]].

## 왜 나눴나 (workspace 하나로 다 못 하는 이유)

만약 `platform`까지 포함해서 전부 workspace로만 나누면, `terraform workspace select release && apply` / `terraform workspace select prod && apply`를 각각 돌릴 때마다 **VPC도 EKS도 워크스페이스별로 따로 생긴다** — release용 EKS, prod용 EKS 두 개가 되어버려서 EKS 컨트롤플레인 비용이 거의 2배가 되고, 애초에 "VPC/EKS는 공유"라는 요구사항에 안 맞는다. 그래서 공유해야 하는 것(`platform`)과 환경별로 달라야 하는 것(`workload`)을 root 자체를 나누는 걸로 분리하고, `workload` 안에서만 workspace를 쓴다.

## `workload` 안에서 release/prod를 나누는 법: terraform workspace

```bash
cd Infra/terraform/workload
terraform workspace new release   # 최초 1회
terraform workspace new prod      # 최초 1회
terraform workspace select release
terraform apply
```

S3 backend + workspace 조합이면 state key가 자동으로 `env:/<workspace>/workload/terraform.tfstate`로 나뉘어서, release/prod가 물리적으로 완전히 다른 state 파일에 저장된다.

## `default` workspace 실수 방지: `workspace_guard`

`terraform init` 직후엔 항상 `default`라는 workspace에 있다. 여기서 실수로 `apply`하면 안 되므로 `workload/main.tf`에 이런 가드를 넣어뒀다:

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

`db_instance_class`, `multi_az`, `skip_final_snapshot`, `deletion_protection`, `redis_node_type`, `force_destroy` 같은 값들은 release/prod가 달라야 한다. `workload/main.tf`의 locals에 workspace-키 맵으로 정의:

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

## 관련
- [[terraform-module-boundaries]]
- [[terraform-remote-state]]
- [[2026-08-06-terraform-module-restructure]]
