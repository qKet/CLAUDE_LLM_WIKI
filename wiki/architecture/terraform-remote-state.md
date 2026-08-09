---
title: Terraform state — S3 backend와 remote_state 참조 구조
category: architecture
tags: [infra, terraform]
created: 2026-08-06
updated: 2026-08-06
---

# Terraform state — S3 backend와 remote_state 참조 구조

## 배경: Terraform state가 뭔지

Terraform은 `apply`할 때마다 "지금 AWS에 뭐가 실제로 만들어져 있는지"를 `terraform.tfstate`라는 JSON 파일에 기록한다. 다음 `plan`/`apply` 때 이 파일을 보고 이미 있는 건 안 만들고 바뀐 것만 반영한다.

기본값은 로컬 디스크에 저장하는 것이지만, 팀 프로젝트에서는 팀원마다 다른 state를 갖게 되는 문제(충돌·유실)가 생기므로 **S3 같은 원격 저장소에 state를 두고 공유**한다. 이게 `backend "s3" { ... }` 설정이다.

> ⚠️ 헷갈리기 쉬운 점: 이 S3 버킷(`team5-qket-tfstate-727646470302`)은 **Terraform 자기 자신의 작업 기록 보관소**일 뿐, 앱이 쓰는 리소스가 아니다. 포스터 이미지를 담는 S3 버킷([[terraform-module-boundaries|storage 모듈]]이 만드는 것, 이름이 `team5-qket-posters-release` 같은 형태)과는 완전히 별개다.

## Qket의 구조: platform / workload 두 root, state도 분리

`Infra/terraform`은 root가 두 개다(자세한 모듈 경계는 관련 문서 참고):

- `platform/` — VPC/서브넷/EKS/bastion/ECR(공유 싱글턴), 자기 state를 `platform/terraform.tfstate` 키로 S3에 저장
- `workload/` — RDS/Redis/storage(release/prod별로 workspace로 분리), 자기 state를 `workload/terraform.tfstate` 키로 저장

```
team5-qket-tfstate-727646470302/            (S3 버킷, 둘이 공유)
├── platform/terraform.tfstate               ← platform/backend.tf가 여기에 씀
└── workload/terraform.tfstate                ← workload/backend.tf가 여기에 씀 (workspace별로 추가 분리됨)
```

## workload가 platform의 값을 가져다 쓰는 법: `terraform_remote_state`

`workload`는 RDS/Redis를 어느 VPC·서브�트에 넣을지 알아야 하는데, 그 정보는 `platform`이 만들었으므로 `workload`의 state엔 없다. 그래서 `workload/remote-state.tf`에 이렇게 선언한다:

```hcl
data "terraform_remote_state" "platform" {
  backend = "s3"
  config = {
    bucket = "team5-qket-tfstate-727646470302"
    key    = "platform/terraform.tfstate"
    region = "ap-northeast-2"
  }
}
```

그러면 `main.tf` 등에서 이렇게 쓸 수 있다:

```hcl
private_data_subnet_ids       = data.terraform_remote_state.platform.outputs.private_data_subnet_ids
eks_cluster_security_group_id = data.terraform_remote_state.platform.outputs.eks_cluster_security_group_id
```

## 중요한 디테일: state 전체가 아니라 output만 노출됨

S3에 있는 `platform/terraform.tfstate`는 platform이 만든 **모든 리소스의 전체 정보**(각 리소스의 모든 attribute)를 담은 큰 JSON이다. 하지만 `terraform_remote_state`로 접근하면 그 전체가 아니라, **platform이 `output "xxx" { value = ... }`으로 명시적으로 내보낸 값들만** `.outputs.xxx`로 읽을 수 있다.

즉 `platform/outputs.tf`가 사실상 **API 계약** 역할을 한다 — platform이 내부 리소스 이름을 리팩터링해도 output 인터페이스만 안 바뀌면 `workload`는 영향을 안 받는다.

## 비유

`platform`이 "이 값들 공개합니다"라고 대문에 게시판(output)을 붙여두면, `workload`는 그 게시판을 읽는 창구(`remote_state`) 역할만 한다. `workload`가 직접 VPC나 EKS를 만들 권한은 없고, 오직 읽기만 한다.

## 관련
- [[README|decisions/README]]
