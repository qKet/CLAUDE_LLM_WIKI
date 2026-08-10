---
title: Terraform 최초 적용 절차 (infrastructure → k8s-addon → registry/data)
category: runbook
tags: [infra, terraform]
created: 2026-08-06
updated: 2026-08-10
---

# Terraform 최초 적용 절차 (infrastructure → k8s-addon → registry/data)

> ⚠️ 2026-08-10에 root 이름이 `platform`→`infrastructure`, `workload`→`data`로 바뀌었고, apply 순서를 그대로 드러내는 **숫자 접두사**가 붙었다: 실제 디렉토리명은 `01_infrastructure`/`02_k8s-addon`/`03_registry`/`04_data`. 원래 `infrastructure`에 같이 있던 `kubernetes_namespace.qket`/`helm_release.argocd`는 `02_k8s-addon`으로, `module.ecr`/`module.github_actions_oidc`는 `03_registry`로 각각 분리되면서 **root가 2개에서 4개로 늘었다**. 본문에서는 편의상 접두사 없이 `infrastructure`/`k8s-addon`/`registry`/`data`로 부르되, 실제 명령어(`cd` 등)는 숫자 접두사를 붙인 진짜 경로로 적는다.

## 배경

네 root가 있고, 그중 셋(`infrastructure`/`k8s-addon`/`data`)은 순서대로 서로를 의존한다. `registry`는 독립적이라 아무 때나 적용해도 된다:

- `infrastructure`(VPC/EKS/bastion) — 순수 AWS 리소스만, kubernetes/helm provider 안 씀. 맨 먼저 apply.
- `k8s-addon`(namespace/ArgoCD) — `infrastructure`의 output을 [[terraform-remote-state|원격으로 참조]]해서 EKS API에 인증(kubernetes/helm provider). `infrastructure` 다음, `data`보다 먼저 apply.
- `registry`(ECR/github-actions-oidc) — 다른 root와 의존관계 없음(EKS/VPC 필요 없음). 순서 상관없이 아무 때나 apply 가능.
- `data`(RDS/Redis/storage, release/prod) — `infrastructure`의 output(VPC/서브넷 등)과 `k8s-addon`이 만든 네임스페이스(`qket-release`/`qket-prod`)가 이미 있다는 걸 전제로 `kubernetes_config_map`/`module.storage`(ServiceAccount)를 만듦. `infrastructure`/`k8s-addon` 다음에 apply.

`data`가 네임스페이스를 직접 안 만드는 이유: `data`는 workspace라서 release/prod를 나눠서 두 번 apply해야 하는데, 네임스페이스는 `data`의 첫 apply 시점부터 이미 있어야 한다(`kubernetes_config_map`/`module.storage`가 그 존재를 전제). 그래서 workspace 없이 한 번만 apply되는 `k8s-addon`이 대신 둘 다 미리 만들어둔다. (원래는 ArgoCD가 `Infra/kubernetes/*.yaml`로 네임스페이스까지 관리할 계획이었는데, 그 ArgoCD Application이 아직 없어서 새 클러스터마다 `data` apply가 "namespace not found"로 실패하는 걸 겪고 이렇게 옮김 — 자세한 경위는 [[../troubleshooting/eks-destroy-layer-separation]] 참고.)

## 절차

### 1. infrastructure 먼저 적용 (한 번만, workspace 없음)

```bash
cd Infra/01_infrastructure
terraform init
terraform plan
terraform apply
```

### 2. k8s-addon 적용 (한 번만, workspace 없음)

```bash
cd Infra/02_k8s-addon
terraform init
terraform plan
terraform apply
```

### 3. registry 적용 (한 번만, workspace 없음 — 다른 root와 순서 무관, 아무 때나 해도 됨)

```bash
cd Infra/03_registry
terraform init
terraform plan
terraform apply
```

### 4. data workspace 생성 (최초 1회)

```bash
cd Infra/04_data
terraform init
terraform workspace new release
terraform workspace new prod
```

### 5. release 적용

```bash
terraform workspace select release
terraform plan
terraform apply
```

### 6. prod 적용

```bash
terraform workspace select prod
terraform plan
terraform apply
```

## 주의사항

- `data`에서 `terraform workspace select`를 안 하고 그냥 `apply`하면 `default` workspace로 잡히는데, [[terraform-platform-workload-split|workspace_guard]]가 명확한 에러 메시지로 막아준다 — 에러가 나면 당황하지 말고 `terraform workspace select release`(또는 `prod`) 하고 다시 시도.
- `infrastructure`를 나중에 변경(예: VPC CIDR 수정)하면 `k8s-addon`/`data`가 참조하는 값이 바뀔 수 있으니, `infrastructure` 변경 후엔 둘 다 `plan`으로 영향 확인. `registry`는 `infrastructure`와 무관하니 영향 없음.
- 클러스터를 destroy 후 재생성하는 경우, `infrastructure`→`k8s-addon` apply가 끝나야 EKS와 네임스페이스가 다시 생기므로 `data`를 먼저/동시에 시도하면 위 이유로 실패한다 — 반드시 순서대로 기다렸다 넘어갈 것. `registry`는 이 사이클과 무관하니 신경 안 써도 됨.
- **destroy 순서는 반대**: `data`(원칙적으로 안 지움) → `k8s-addon` → `infrastructure`. `registry`는 원칙적으로 안 지움. `infrastructure`의 bastion SG를 `data`의 rds/redis SG가 CIDR로만 참조하도록 바꿔놔서(SG ID 직접 참조 안 함) `data`↔`infrastructure` 사이엔 이제 DependencyViolation이 안 나지만, `k8s-addon`은 EKS Access Entry가 살아있는 동안(=`infrastructure`를 지우기 전) 먼저 지워야 정상적으로 인증돼서 지워짐 — 자세한 내용은 [[../troubleshooting/eks-destroy-layer-separation]].
- **매일 아침/저녁 `infrastructure`만 껐다 켜는 절차는 이 문서(최초 적용)와 다름** — [[daily-infrastructure-toggle]] 참고. VPC/서브넷/`registry`는 그대로 두고 EKS/bastion/NAT만 골라서 지웠다 켬.
- 2026-08-10 기준: root 이름 변경 + 기존 AWS 리소스 전체 destroy + `k8s-addon`/`registry` root 신설이 전부 끝난 상태 — **AWS 계정엔 아직 아무것도 없고(빈 상태), 넷 다 `terraform validate`만 통과 확인했지 `apply`는 안 함.** 다음 apply가 이 절차 그대로 처음 실행하는 것이 된다.

## 관련
- [[terraform-remote-state]]
- [[terraform-platform-workload-split]]
- [[../troubleshooting/eks-destroy-layer-separation]]
- [[daily-infrastructure-toggle]]
