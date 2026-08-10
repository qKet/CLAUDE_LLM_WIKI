---
title: Terraform 최초 적용 절차 (infrastructure → data)
category: runbook
tags: [infra, terraform]
created: 2026-08-06
updated: 2026-08-10
---

# Terraform 최초 적용 절차 (infrastructure → data)

> ⚠️ 2026-08-10에 root 이름이 `platform`→`infrastructure`, `workload`→`data`로 바뀌었고, 곧이어 apply 순서를 그대로 드러내는 **숫자 접두사**가 붙었다: 실제 디렉토리명은 `01_infrastructure`/`04_data`(`02_k8s-addon`/`03_registry`는 구상 단계, 아직 없음). 본문에서는 편의상 접두사 없이 `infrastructure`/`data`로 부르되, 실제 명령어(`cd` 등)는 숫자 접두사를 붙인 진짜 경로로 적는다.

## 배경

`infrastructure`(VPC/EKS/bastion/ECR)과 `data`(RDS/Redis/storage, release/prod)가 별도 root라서 순서를 지켜야 한다 — `data`가 `infrastructure`의 output을 [[terraform-remote-state|원격으로 참조]]하기 때문에, `infrastructure`가 먼저 apply되어 있어야 `data`가 필요한 값을 읽을 수 있다.

`infrastructure`는 `qket-release`/`qket-prod` 네임스페이스도 같이 만든다(`kubernetes_namespace.qket`, `for_each`로 둘 다 한 번에) — `data`의 `kubernetes_config_map`/`module.storage`(ServiceAccount 등)가 그 네임스페이스가 이미 있다고 전제하기 때문에, 순서상 `infrastructure`가 먼저 끝나 있어야 하는 이유가 하나 더 늘었다. (원래는 ArgoCD가 `Infra/kubernetes/*.yaml`로 네임스페이스까지 관리할 계획이었는데, 그 ArgoCD Application이 아직 없어서 새 클러스터마다 `data` apply가 "namespace not found"로 실패하는 걸 겪고 `infrastructure`로 옮김 — 자세한 경위는 `01_infrastructure/main.tf`의 `kubernetes_namespace.qket` 주석, [[../troubleshooting/eks-destroy-layer-separation]] 참고.)

> ⚠️ 이 `kubernetes_namespace.qket`/`helm_release.argocd`는 원래 계획대로면 `infrastructure`가 아니라 별도 `k8s-addon` root(구상 단계)에 있어야 한다 — 아직 이전 전이라 지금은 `infrastructure`에 같이 있다. [[../troubleshooting/eks-destroy-layer-separation]] 참고.

## 절차

### 1. infrastructure 먼저 적용 (한 번만, workspace 없음)

```bash
cd Infra/01_infrastructure
terraform init
terraform plan
terraform apply
```

### 2. data workspace 생성 (최초 1회)

```bash
cd Infra/04_data
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

- `data`에서 `terraform workspace select`를 안 하고 그냥 `apply`하면 `default` workspace로 잡히는데, [[terraform-platform-workload-split|workspace_guard]]가 명확한 에러 메시지로 막아준다 — 에러가 나면 당황하지 말고 `terraform workspace select release`(또는 `prod`) 하고 다시 시도.
- `infrastructure`를 나중에 변경(예: VPC CIDR 수정)하면 `data`가 참조하는 값이 바뀔 수 있으니, `infrastructure` 변경 후엔 `data`도 release/prod 양쪽 다 `plan`으로 영향 확인.
- 클러스터를 destroy 후 재생성하는 경우, `infrastructure` apply가 끝나야 네임스페이스가 다시 생기므로 `data`를 먼저/동시에 시도하면 위 이유로 실패한다 — 반드시 `infrastructure` apply 완료를 기다린 뒤 `data`로 넘어갈 것.
- **destroy 순서는 반대**: `data`(원칙적으로 안 지움) → `infrastructure`. `infrastructure`의 bastion SG를 `data`의 rds/redis SG가 CIDR로만 참조하도록 바꿔놔서(SG ID 직접 참조 안 함) 이제 순서를 안 지켜도 DependencyViolation은 안 나지만, kubernetes/helm provider 리소스(`helm_release.argocd`, `kubernetes_namespace.qket`) 때문에 여전히 `data`를 먼저 지우는 걸 권장 — 자세한 내용은 [[../troubleshooting/eks-destroy-layer-separation]].
- 2026-08-10 기준: root 이름 변경 + `data`/`infrastructure` 전체를 한 번 완전히 destroy해서 **지금은 AWS 리소스가 하나도 없는 완전히 빈 상태**다. 다음 apply가 새 이름(`infrastructure`/`data`)으로 하는 첫 apply가 된다 — `data`의 `release` workspace도 다시 `terraform workspace select release`(또는 `new`)부터 해야 함.

## 관련
- [[terraform-remote-state]]
- [[terraform-platform-workload-split]]
- [[../troubleshooting/eks-destroy-layer-separation]]
