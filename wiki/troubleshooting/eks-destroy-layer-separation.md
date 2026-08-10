---
title: EKS 인프라 destroy 시 K8s addon(ArgoCD/namespace/ingress controller) 정리 실패 — 근본 원인과 Layer 분리 해법
category: troubleshooting
tags: [infra, terraform, eks, k8s, argocd]
created: 2026-08-10
updated: 2026-08-10
---

# EKS 인프라 destroy 시 K8s addon 정리 실패 — 근본 원인과 Layer 분리 해법

> ⚠️ 이 문서의 "해결"은 **아직 코드로 실제 구현되지 않은, 결정만 된 상태**다. 지금(2026-08-10)도 `platform/main.tf`에 `helm_release.argocd`/`kubernetes_namespace.qket`가 그대로 섞여 있다. "현재 구현 상태" 섹션에 뭐가 됐고 안 됐는지 정확히 남겨둔다.

## 배경

`Infra/platform`(VPC/EKS/bastion/ECR/OIDC + Nginx/ALB 같은 Ingress Controller + ArgoCD + namespace)을 **단일 Terraform state, 단일 apply**로 한 번에 구성했다. 처음 만들 때는(리소스가 늘어나기만 하니까) 문제가 없었는데, `terraform destroy`로 지우려고 하면서부터 매번 다른 형태의 에러가 반복됐다.

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
`platform`의 bastion 보안그룹을 지우려는데, 별도 state인 `workload`의 RDS/Redis 보안그룹이 그 bastion SG를 ingress 규칙의 소스로 아직 참조하고 있어서 AWS가 거부. 자세한 원인/해결(CIDR 참조로 전환)은 [[../architecture/terraform-platform-workload-split]].

**3. (아직 안 겪었지만 같은 계열 — 미리 남겨둠) Ingress Controller가 동적으로 만든 AWS 리소스 잔존**

지금은 Ingress Controller(ALB Controller 등)가 `Infra/backup/`에 보류 중이라 실제로 겪은 적은 없지만, 설치하면 100% 재현될 문제라 미리 적어둔다: Ingress Controller가 뜨면 Kubernetes Ingress 오브젝트를 보고 **Terraform이 모르는 AWS 리소스**(ALB/NLB, Target Group, 전용 보안그룹)를 스스로 만든다. `terraform destroy`는 자기 state에 기록된 것만 지우려고 하는데, 이 ALB가 서브넷의 ENI를 점유하고 있으면 VPC/서브넷 삭제가 `DependencyViolation`으로 막힌다 — helm으로 **먼저 uninstall**해서 ALB 자체가 AWS 쪽에서 정리되게 하지 않으면 반드시 겪는다.

## 원인 (두 갈래로 다른 문제)

| | 원인 | 왜 생기나 |
|---|---|---|
| 증상 1 | **EKS API 접근 수단이 먼저 사라짐** | kubernetes/helm provider가 인증에 쓰는 EKS Access Entry가 K8s 리소스들과 **같은 state** 안에 있어서, destroy 순서가 안 지켜지면 인증 수단부터 없어짐 |
| 증상 2·3 | **다른 곳이 만든 AWS 리소스가 아직 참조/점유 중** | (2) 다른 state(`workload`)가 SG ID를 직접 참조 / (3) K8s가 Terraform 밖에서 동적으로 만든 리소스(ALB)가 서브넷을 점유 — 둘 다 "Terraform이 모르는 곳에서 걸린 의존"이라는 공통점 |

즉 겉보기엔 비슷한 "destroy가 안 된다"는 증상이지만, 하나는 **인증 순서** 문제고 다른 하나는 **AWS 레벨 참조** 문제라 해법도 따로 필요하다.

## 해결

### 임시 조치 (지금 당장 급할 때)

- 증상 1: `terraform destroy -target=helm_release.argocd -target='kubernetes_namespace.qket'` 로 kubernetes-provider 리소스만 먼저 지운 뒤 나머지 destroy (EKS가 아직 살아있어서 정상 인증됨)
- 증상 2: `workload`를 `platform`보다 먼저 destroy(또는 SG 참조 자체를 없앰 — 아래 근본 해결 참고)
- 증상 3: Ingress Controller가 설치돼 있다면 helm release부터 먼저 `-target`으로 uninstall

### 근본 해결: Layer(=root) 자체를 분리

```
Infra/
├── platform/           Layer 1 — VPC, EKS(cluster+node group), bastion, ECR, OIDC
│                        순수 AWS API 리소스만. kubernetes/helm provider 안 씀.
│                        destroy할 땐 AWS API만 호출하니 순서 꼬임 없이 원샷으로 지워짐.
│
├── cluster-bootstrap/   Layer 2 — namespace, ArgoCD, Ingress Controller 등 K8s addon
│                        terraform_remote_state로 platform의 output(EKS 이름/엔드포인트/
│                        CA/cluster_admin_role_arn)을 받아서 kubernetes/helm provider로 배포.
│
└── workload/            RDS/Redis/S3 — platform의 SG를 CIDR로만 참조(SG ID 직접 참조 안 함).
```

- **apply 순서**: `platform` → `cluster-bootstrap` → `workload`
- **destroy 순서**: `workload`(원칙적으로 안 지움) → `cluster-bootstrap`(EKS가 아직 살아있을 때 K8s addon부터 깨끗하게 정리) → `platform`(이제 아무도 안 걸고 있으니 AWS 리소스만 원샷 삭제)

이렇게 Layer 1(순수 AWS)과 Layer 2(K8s addon)를 state 자체로 분리하면:
- 증상 1: Layer 2를 Layer 1보다 먼저 지우는 게 **디렉토리 구조 자체로 강제**되므로 인증 수단이 먼저 사라질 일이 없음
- 증상 3(예방): Ingress Controller도 Layer 2에 있으므로, Layer 2 destroy 시점에 EKS가 살아있어 helm uninstall이 정상 동작 → ALB가 제대로 정리된 뒤에 Layer 1이 서브넷을 지움

## 현재 구현 상태 (2026-08-10 기준, 정직하게)

- ✅ **결정은 확정**: 이 문서의 Layer 1/Layer 2 분리를 채택하기로 함
- ✅ 증상 2(SG 참조)는 코드 수정 완료 — `workload`가 CIDR 기반으로 전환됨(단, 아직 `terraform apply` 전)
- ❌ **`cluster-bootstrap` root 자체는 아직 안 만들어짐** — `helm_release.argocd`/`kubernetes_namespace.qket`가 여전히 `platform/main.tf`에 같이 있음
- ❌ Ingress Controller는 아직 `Infra/backup/`에 보류 중, 아직 어느 root에도 설치 안 됨
- 지금 당장은 "임시 조치"(위 `-target` 방식)로 버티고 있는 상태

## 관련
- [[eks-provider-auth]]
- [[../architecture/terraform-platform-workload-split]]
- [[../runbook/terraform-apply-order]]
