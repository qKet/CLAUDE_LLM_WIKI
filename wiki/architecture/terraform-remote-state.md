---
title: Terraform state — S3 backend와 remote_state 참조 구조
category: architecture
tags: [infra, terraform]
created: 2026-08-06
updated: 2026-08-10
---

# Terraform state — S3 backend와 remote_state 참조 구조

> ⚠️ 2026-08-10에 root 이름이 `platform`→`infrastructure`, `workload`→`data`로 바뀌었다(기능 동일, 이름만 변경). 이 문서는 새 이름 기준.

## 배경: Terraform state가 뭔지

Terraform은 `apply`할 때마다 "지금 AWS에 뭐가 실제로 만들어져 있는지"를 `terraform.tfstate`라는 JSON 파일에 기록한다. 다음 `plan`/`apply` 때 이 파일을 보고 이미 있는 건 안 만들고 바뀐 것만 반영한다.

기본값은 로컬 디스크에 저장하는 것이지만, 팀 프로젝트에서는 팀원마다 다른 state를 갖게 되는 문제(충돌·유실)가 생기므로 **S3 같은 원격 저장소에 state를 두고 공유**한다. 이게 `backend "s3" { ... }` 설정이다.

> ⚠️ 헷갈리기 쉬운 점: 이 S3 버킷(`team5-qket-tfstate-727646470302`)은 **Terraform 자기 자신의 작업 기록 보관소**일 뿐, 앱이 쓰는 리소스가 아니다. 포스터 이미지를 담는 S3 버킷([[terraform-module-boundaries|storage 모듈]]이 만드는 것, 이름이 `team5-qket-posters-release` 같은 형태)과는 완전히 별개다.

## Qket의 구조: infrastructure / data 두 root, state도 분리

`Infra`는 root가 두 개다(자세한 모듈 경계는 관련 문서 참고):

- `infrastructure/`(구 `platform`) — VPC/서브넷/EKS/bastion/ECR(공유 싱글턴), 자기 state를 `infrastructure/terraform.tfstate` 키로 S3에 저장
- `data/`(구 `workload`) — RDS/Redis/storage(release/prod별로 workspace로 분리), 자기 state를 `data/terraform.tfstate` 키로 저장

```
team5-qket-tfstate-727646470302/            (S3 버킷, 둘이 공유)
├── infrastructure/terraform.tfstate         ← infrastructure/backend.tf가 여기에 씀
└── data/terraform.tfstate                    ← data/backend.tf가 여기에 씀 (workspace별로 추가 분리됨)
```

## data가 infrastructure의 값을 가져다 쓰는 법: `terraform_remote_state`

`data`는 RDS/Redis를 어느 VPC·서브넷에 넣을지 알아야 하는데, 그 정보는 `infrastructure`가 만들었으므로 `data`의 state엔 없다. 그래서 `data/remote-state.tf`에 이렇게 선언한다:

```hcl
data "terraform_remote_state" "infrastructure" {
  backend = "s3"
  config = {
    bucket = "team5-qket-tfstate-727646470302"
    key    = "infrastructure/terraform.tfstate"
    region = "ap-northeast-2"
  }
}
```

그러면 `main.tf` 등에서 이렇게 쓸 수 있다:

```hcl
private_data_subnet_ids       = data.terraform_remote_state.infrastructure.outputs.private_data_subnet_ids
eks_cluster_security_group_id = data.terraform_remote_state.infrastructure.outputs.eks_cluster_security_group_id
```

## 중요한 디테일: state 전체가 아니라 output만 노출됨

S3에 있는 `infrastructure/terraform.tfstate`는 infrastructure가 만든 **모든 리소스의 전체 정보**(각 리소스의 모든 attribute)를 담은 큰 JSON이다. 하지만 `terraform_remote_state`로 접근하면 그 전체가 아니라, **infrastructure가 `output "xxx" { value = ... }`으로 명시적으로 내보낸 값들만** `.outputs.xxx`로 읽을 수 있다.

즉 `infrastructure/outputs.tf`가 사실상 **API 계약** 역할을 한다 — infrastructure가 내부 리소스 이름을 리팩터링해도 output 인터페이스만 안 바뀌면 `data`는 영향을 안 받는다.

> ⚠️ 실제로 겪은 함정: `infrastructure`를 destroy하다가 중간에 멈추면(예: SG DependencyViolation), 이미 지워진 리소스를 참조하는 output이 있을 경우 **state의 output이 통째로 비어버릴 수 있다** — 이러면 `data`가 `terraform_remote_state`를 읽으려 할 때 "object with no attributes"로 실패한다. `terraform apply -refresh-only`로 다시 채워야 함. 자세한 사례는 [[../troubleshooting/eks-destroy-layer-separation]].

## 비유

`infrastructure`가 "이 값들 공개합니다"라고 대문에 게시판(output)을 붙여두면, `data`는 그 게시판을 읽는 창구(`remote_state`) 역할만 한다. `data`가 직접 VPC나 EKS를 만들 권한은 없고, 오직 읽기만 한다.

## 관련
- [[README|decisions/README]]
- [[../troubleshooting/eks-destroy-layer-separation]]
