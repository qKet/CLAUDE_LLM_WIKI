---
title: Terraform 모듈 경계 (vpc/subnet/security_group/ec2/eks/rds/redis/storage/ecr)
category: architecture
tags: [infra, terraform]
created: 2026-08-06
updated: 2026-08-10
---

# Terraform 모듈 경계

`Infra/terraform/modules/`는 리소스 타입 하나당 모듈 하나로 나뉘어 있다. 각 모듈은 `main.tf`/`variables.tf`/`outputs.tf` 3파일 구성으로 통일.

## 모듈 목록과 소유 리소스

| 모듈 | 소유 리소스 | 비고 |
|---|---|---|
| `vpc` | `aws_vpc`, `aws_internet_gateway` | subnet 쪽 출력을 전혀 필요로 하지 않음(중요, 아래 참고) |
| `subnet` | `aws_subnet`(public/private), `aws_eip`+`aws_nat_gateway`, `aws_route_table`+연결 | `vpc_id`/`igw_id`를 입력으로만 받음 — 단방향 |
| `security_group` | `aws_security_group` (범용, `for_each`) | 아래 별도 설명 |
| `ec2` | bastion용 `aws_instance`+IAM role/profile | SG는 안 만들고 `security_group_id`를 입력받음 |
| `eks` | `aws_eks_cluster`, `aws_eks_node_group`, IAM role, OIDC provider | 기존과 동일, 변경 없음 |
| `rds` | `aws_db_subnet_group`, `aws_db_instance` | SG는 안 만듦. `multi_az`/`skip_final_snapshot`/`deletion_protection`을 변수로 받음(아래 [[terraform-platform-workload-split]] 참고) |
| `redis` | `aws_elasticache_subnet_group`, `aws_elasticache_cluster` | SG는 안 만듦 |
| `storage` | S3(포스터 이미지)+CloudFront+백엔드 IRSA+k8s ServiceAccount/ConfigMap | 기존과 동일. `force_destroy`를 변수로 받음 |
| `ecr` | `aws_ecr_repository`+lifecycle policy | 신규. untagged 이미지 7일 후 자동 만료 |

`network`(vpc+subnet 합쳐져 있던 것)와 `data`(rds+redis 합쳐져 있던 것) 모듈은 이 재편으로 없어졌다. 재편 배경은 [[2026-08-06-terraform-module-restructure]] 참고.

## `security_group`을 범용 다중-SG 팩토리로 만든 이유

원래는 SG가 각 모듈(bastion의 `aws_security_group.ssm_bastion`, data의 `aws_security_group.rds`/`aws_security_group.redis`)에 흩어져 inline으로 있었다. 지금은 `for_each`로 여러 SG를 한 번에 만드는 범용 모듈 하나로 뽑아냈다:

```hcl
module "security_group" {
  source = "../modules/security_group"
  vpc_id = ...
  security_groups = {
    bastion = { ingress = [] }
    # 또는 data에서:
    "rds-${local.environment}" = { ingress = [...] }
  }
}
```

`infrastructure`에서 한 번(bastion SG), `data`에서 한 번(rds/redis SG) 각각 호출한다 — 층이 다르니 모듈 자체를 두 곳에서 재사용하는 구조.

> ⚠️ 모든 SG에 아웃바운드 전체 허용(`egress` 전체) 규칙을 동일하게 넣어뒀다. 원래 bastion만 이랬는데 rds/redis도 이렇게 통일함 — `aws_security_group`은 inline `egress` 블록을 아예 안 쓰면 AWS가 기본으로 만드는 outbound 규칙까지 Terraform이 지워버리는 잘 알려진 함정이 있어서, 명시적으로 항상 선언해뒀다.

## ECR을 `infrastructure`에 둔 이유 (data 아님)

backend/frontend 이미지는 release/prod가 **태그로만 구분되는 하나의 공유 저장소**를 쓴다(예: `backend-<sha>`). `data`는 terraform workspace로 release/prod를 나누는데, 만약 `ecr` 모듈을 `data`에 넣으면 workspace마다 별도 ECR 저장소가 생겨버려서(태그 기반 공유라는 전제가 깨짐) `infrastructure`(공유 싱글턴 계층)에 뒀다.

> ⚠️ 원래 계획대로면 ECR은 `infrastructure`도 아니고 절대 안 지우는 별도 `registry` root(구상 단계)에 있어야 한다 — `infrastructure`는 매일 켜고 끄는 대상이라 ECR을 같이 두는 게 위험하기 때문. 아직 이전 안 함, [[../troubleshooting/eks-destroy-layer-separation]] 참고.

## 관련
- [[terraform-platform-workload-split]]
- [[../troubleshooting/eks-destroy-layer-separation]]
- [[terraform-remote-state]]
- [[2026-08-06-terraform-module-restructure]]
