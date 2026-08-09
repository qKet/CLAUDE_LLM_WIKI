---
title: Terraform 최초 적용 절차 (platform → workload)
category: runbook
tags: [infra, terraform]
created: 2026-08-06
updated: 2026-08-06
---

# Terraform 최초 적용 절차 (platform → workload)

## 배경

`platform`(VPC/EKS/bastion/ECR)과 `workload`(RDS/Redis/storage, release/prod)가 별도 root라서 순서를 지켜야 한다 — `workload`가 `platform`의 output을 [[terraform-remote-state|원격으로 참조]]하기 때문에, `platform`이 먼저 apply되어 있어야 `workload`가 필요한 값을 읽을 수 있다.

## 절차

### 1. platform 먼저 적용 (한 번만, workspace 없음)

```bash
cd Infra/terraform/platform
terraform init
terraform plan
terraform apply
```

### 2. workload workspace 생성 (최초 1회)

```bash
cd Infra/terraform/workload
terraform init
terraform workspace new release
terraform workspace new prod
```

### 3. release 적용

```bash
terraform workspace select release
terraform plan
terraform apply
```

### 4. prod 적용

```bash
terraform workspace select prod
terraform plan
terraform apply
```

## 주의사항

- `workload`에서 `terraform workspace select`를 안 하고 그냥 `apply`하면 `default` workspace로 잡히는데, [[terraform-platform-workload-split|workspace_guard]]가 명확한 에러 메시지로 막아준다 — 에러가 나면 당황하지 말고 `terraform workspace select release`(또는 `prod`) 하고 다시 시도.
- `platform`을 나중에 변경(예: VPC CIDR 수정)하면 `workload`가 참조하는 값이 바뀔 수 있으니, `platform` 변경 후엔 `workload`도 release/prod 양쪽 다 `plan`으로 영향 확인.
- 지금(2026-08-06 기준) 아직 실제로 한 번도 apply된 적 없는 상태 — 처음 적용할 때 이 문서 자체도 실제 경험 기반으로 업데이트할 것.

## 관련
- [[terraform-remote-state]]
- [[terraform-platform-workload-split]]
