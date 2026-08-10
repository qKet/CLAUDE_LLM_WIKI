---
title: EKS 인프라 destroy 시 K8s addon(ArgoCD/namespace/ingress controller) 정리 실패 — 근본 원인과 Layer 분리 해법
category: troubleshooting
tags: [infra, terraform, eks, k8s, argocd]
created: 2026-08-10
updated: 2026-08-10
---

# EKS 인프라 destroy 시 K8s addon 정리 실패 — 근본 원인과 Layer 분리 해법

> ⚠️ root 이름이 `platform`→`infrastructure`, `workload`→`data`로 바뀌었고(2026-08-10, 기능 동일), 이 문서가 제안하는 Layer 2 root의 이름도 `cluster-bootstrap`이 아니라 **`k8s-addon`**으로 정했다. 이 문서는 새 이름 기준으로 갱신했다.
>
> ✅ 2026-08-10: 이 문서의 "해결"(Layer 분리)이 실제로 구현됐다 — `helm_release.argocd`/`kubernetes_namespace.qket`가 `02_k8s-addon`으로 이전 완료. "현재 구현 상태" 섹션 참고.

## 배경

`Infra/01_infrastructure`(구 `platform`; VPC/EKS/bastion/ECR/OIDC + Nginx/ALB 같은 Ingress Controller + ArgoCD + namespace)을 **단일 Terraform state, 단일 apply**로 한 번에 구성했다. 처음 만들 때는(리소스가 늘어나기만 하니까) 문제가 없었는데, `terraform destroy`로 지우려고 하면서부터 매번 다른 형태의 에러가 반복됐다.

## 증상

**1. kubernetes/helm provider가 쓰는 리소스에서 인증 실패**
```
Error: Kubernetes cluster unreachable: the server has asked for the client to provide credentials
Error: Unauthorized
```
`helm_release.argocd`, `kubernetes_namespace.qket["release"]`, `kubernetes_namespace.qket["prod"]`를 지우려 할 때 발생. 자세한 원인/재현은 [[eks-provider-auth]].

**2. AWS 리소스가 아직 참조되고 있다며 삭제 거부**
```
Error: deleting Security Group (sg-0534a94c1418e8c06): ... DependencyViolation:
resource sg-0534a94c1418e8c06 has a dependent object
```
`infrastructure`의 bastion 보안그룹을 지우려는데, 별도 state인 `data`의 RDS/Redis 보안그룹이 그 bastion SG를 ingress 규칙의 소스로 아직 참조하고 있어서 AWS가 거부. 자세한 원인/해결(CIDR 참조로 전환)은 [[../architecture/terraform-platform-workload-split]].

**3. (아직 안 겪었지만 같은 계열 — 미리 남겨둠) Ingress Controller가 동적으로 만든 AWS 리소스 잔존**

지금은 Ingress Controller(ALB Controller 등)가 `Infra/backup/`에 보류 중이라 실제로 겪은 적은 없지만, 설치하면 100% 재현될 문제라 미리 적어둔다: Ingress Controller가 뜨면 Kubernetes Ingress 오브젝트를 보고 **Terraform이 모르는 AWS 리소스**(ALB/NLB, Target Group, 전용 보안그룹)를 스스로 만든다. `terraform destroy`는 자기 state에 기록된 것만 지우려고 하는데, 이 ALB가 서브넷의 ENI를 점유하고 있으면 VPC/서브넷 삭제가 `DependencyViolation`으로 막힌다 — helm으로 **먼저 uninstall**해서 ALB 자체가 AWS 쪽에서 정리되게 하지 않으면 반드시 겪는다.

**4. (실제로 겪음, 2026-08-10) destroy가 중간에 멈추면 output이 통째로 비어버림**

`infrastructure` destroy가 증상 2(bastion SG DependencyViolation)에서 멈췄을 때, 그 시점에 이미 지워진 리소스(`helm_release.argocd` 등)를 참조하는 output이 있어서 state의 output이 전부 비어버렸다. 그 여파로 `data`가 `terraform_remote_state`로 `infrastructure`의 output을 읽으려다 "object with no attributes"로 실패 — `data`는 이 destroy와 아무 상관이 없었는데도 막힘. `cd 01_infrastructure && terraform apply -refresh-only -auto-approve`로 output을 다시 채워서 해결함.

## 원인 (세 갈래로 다른 문제)

| | 원인 | 왜 생기나 |
|---|---|---|
| 증상 1 | **EKS API 접근 수단이 먼저 사라짐** | kubernetes/helm provider가 인증에 쓰는 EKS Access Entry가 K8s 리소스들과 **같은 state** 안에 있어서, destroy 순서가 안 지켜지면 인증 수단부터 없어짐 |
| 증상 2·3 | **다른 곳이 만든 AWS 리소스가 아직 참조/점유 중** | (2) 다른 state(`data`)가 SG ID를 직접 참조 / (3) K8s가 Terraform 밖에서 동적으로 만든 리소스(ALB)가 서브넷을 점유 — 둘 다 "Terraform이 모르는 곳에서 걸린 의존"이라는 공통점 |
| 증상 4 | **destroy가 중간에 멈추면 output 재계산이 실패** | output이 이미 파괴된 리소스의 attribute를 참조하면, destroy가 도중에 에러로 멈춘 시점의 state 스냅샷엔 그 output들이 아예 비어서 저장됨 — 그 state를 읽는 다른 root(`data`)까지 연쇄로 막힘 |

즉 겉보기엔 비슷한 "destroy가 안 된다"는 증상이지만 원인이 각각 다르다: 인증 순서(1), AWS 레벨 참조(2·3), state/output 재계산(4). 1·2·3은 구조(Layer 분리)로 예방 가능하지만, 4는 destroy가 뭔가에 걸려 멈추는 순간 항상 따라올 수 있는 부작용이라 별도로 인지하고 있어야 한다.

## 해결

### 임시 조치 (지금 당장 급할 때)

- 증상 1: `terraform destroy -target=helm_release.argocd -target='kubernetes_namespace.qket'` 로 kubernetes-provider 리소스만 먼저 지운 뒤 나머지 destroy (EKS가 아직 살아있어서 정상 인증됨)
- 증상 2: `data`를 `infrastructure`보다 먼저 destroy(또는 SG 참조 자체를 없앰 — 아래 근본 해결 참고)
- 증상 3: Ingress Controller가 설치돼 있다면 helm release부터 먼저 `-target`으로 uninstall
- 증상 4: `terraform apply -refresh-only -auto-approve`로 output을 다시 계산해서 state에 씀

### 근본 해결: Layer(=root) 자체를 분리

```
Infra/
├── 01_infrastructure/   Layer 1 — VPC, EKS(cluster+node group), bastion
│                        순수 AWS API 리소스만. kubernetes/helm provider 안 씀.
│                        destroy할 땐 AWS API만 호출하니 순서 꼬임 없이 원샷으로 지워짐.
│                        매일 수동으로 켜고 끄는 대상.
│
├── 02_k8s-addon/        Layer 2 — namespace, ArgoCD, Ingress Controller 등 K8s addon
│                        terraform_remote_state로 infrastructure의 output(EKS 이름/엔드포인트/
│                        CA/cluster_admin_role_arn)을 받아서 kubernetes/helm provider로 배포.
│
├── 03_registry/         ECR, github-actions-oidc — 공유·불변, env 안 나뉨. 절대 안 지움.
│
└── 04_data/             RDS/Redis/S3 — infrastructure의 SG를 CIDR로만 참조(SG ID 직접 참조 안 함).
                         절대 안 지움.
```

숫자 접두사(01~04)는 apply 순서를 그대로 드러내려고 붙임 — Finder/IDE에서 정렬해도 apply해야 하는 순서와 일치한다.

- **apply 순서**: `infrastructure` → `k8s-addon` → `data`/`registry`(둘은 서로 무관, 순서 상관없음)
- **destroy 순서**: `data`(원칙적으로 안 지움) → `k8s-addon`(EKS가 아직 살아있을 때 K8s addon부터 깨끗하게 정리) → `infrastructure`(이제 아무도 안 걸고 있으니 AWS 리소스만 원샷 삭제). `registry`는 원칙적으로 안 지움.

이렇게 Layer 1(순수 AWS)과 Layer 2(K8s addon)를 state 자체로 분리하면:
- 증상 1: Layer 2를 Layer 1보다 먼저 지우는 게 **디렉토리 구조 자체로 강제**되므로 인증 수단이 먼저 사라질 일이 없음
- 증상 3(예방): Ingress Controller도 Layer 2에 있으므로, Layer 2 destroy 시점에 EKS가 살아있어 helm uninstall이 정상 동작 → ALB가 제대로 정리된 뒤에 Layer 1이 서브넷을 지움
- 증상 4(완화): Layer 1(`infrastructure`)의 output이 Layer 2 리소스(`helm_release.argocd` 등)를 더 이상 참조하지 않게 되므로, Layer 1 혼자 destroy될 때 output이 비는 원인 자체가 하나 줄어듦

## 현재 구현 상태 (2026-08-10 기준, 정직하게)

- ✅ **결정은 확정**: 이 문서의 Layer 1/Layer 2 분리 채택, root 이름을 `01_infrastructure`/`02_k8s-addon`/`03_registry`/`04_data`로 확정(apply 순서를 드러내는 숫자 접두사 포함)
- ✅ 증상 2(SG 참조)는 코드 수정 완료 — `04_data`가 CIDR 기반으로 전환됨
- ✅ `platform`/`workload` → `01_infrastructure`/`04_data` 디렉토리 rename 완료(git mv, backend key도 갱신)
- ✅ 기존에 떠있던 AWS 리소스는 전부 destroy 완료 — AWS 계정은 빈 상태였음(마이그레이션 부담 없이 아래 작업 진행)
- ✅ **`02_k8s-addon` root 신설 완료** — `helm_release.argocd`/`kubernetes_namespace.qket`를 `01_infrastructure`에서 이전함. `01_infrastructure/providers.tf`에서 kubernetes/helm/kubectl provider도 완전히 제거 — 이제 순수 AWS API 리소스만 남음. `terraform validate` 셋 다(`01_infrastructure`/`02_k8s-addon`/`04_data`) 통과 확인
- ❌ **`03_registry` root는 아직 안 만들어짐** — ECR/github-actions-oidc는 여전히 `01_infrastructure/main.tf`에 같이 있음
- ❌ Ingress Controller는 아직 `Infra/backup/`에 보류 중, 아직 어느 root에도 설치 안 됨
- 다음 실제 apply는 `01_infrastructure` → `02_k8s-addon` → `04_data` 순서로, 전부 빈 state에서 처음 시작하는 것 — 아직 `terraform apply`는 안 함(코드만 준비됨)

## 관련
- [[eks-provider-auth]]
- [[../architecture/terraform-platform-workload-split]]
- [[../runbook/terraform-apply-order]]
