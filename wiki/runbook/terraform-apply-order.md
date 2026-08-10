---
title: Terraform 최초 적용 절차 (platform → workload)
category: runbook
tags: [infra, terraform]
created: 2026-08-06
updated: 2026-08-10
---

# Terraform 최초 적용 절차 (platform → workload)

## 배경

`platform`(VPC/EKS/bastion/ECR)과 `workload`(RDS/Redis/storage, release/prod)가 별도 root라서 순서를 지켜야 한다 — `workload`가 `platform`의 output을 [[terraform-remote-state|원격으로 참조]]하기 때문에, `platform`이 먼저 apply되어 있어야 `workload`가 필요한 값을 읽을 수 있다.

`platform`은 `qket-release`/`qket-prod` 네임스페이스도 같이 만든다(`kubernetes_namespace.qket`, `for_each`로 둘 다 한 번에) — `workload`의 `kubernetes_config_map`/`module.storage`(ServiceAccount 등)가 그 네임스페이스가 이미 있다고 전제하기 때문에, 순서상 `platform`이 먼저 끝나 있어야 하는 이유가 하나 더 늘었다. (원래는 ArgoCD가 `Infra/kubernetes/*.yaml`로 네임스페이스까지 관리할 계획이었는데, 그 ArgoCD Application이 아직 없어서 새 클러스터마다 `workload` apply가 "namespace not found"로 실패하는 걸 겪고 `platform`으로 옮김 — 자세한 경위는 `platform/main.tf`의 `kubernetes_namespace.qket` 주석 참고.)

## 절차

### 1. platform 먼저 적용 (한 번만, workspace 없음)

```bash
cd Infra/platform
terraform init
terraform plan
terraform apply
```

### 2. workload workspace 생성 (최초 1회)

```bash
cd Infra/workload
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
- 클러스터를 destroy 후 재생성하는 경우, `platform` apply가 끝나야 네임스페이스가 다시 생기므로 `workload`를 먼저/동시에 시도하면 위 이유로 실패한다 — 반드시 `platform` apply 완료를 기다린 뒤 `workload`로 넘어갈 것.
- 2026-08-10 기준 실제로 여러 번 apply/destroy를 거친 상태 — VPC/EKS/bastion/ECR 등 실제 AWS 리소스가 살아있는 게 정상이다.

## 관련
- [[terraform-remote-state]]
- [[terraform-platform-workload-split]]
