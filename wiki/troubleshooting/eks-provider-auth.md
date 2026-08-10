---
title: kubernetes/helm/kubectl provider가 EKS에 인증 못 하던 문제 (토큰 만료 + Unauthorized)
category: troubleshooting
tags: [infra, terraform, eks, iam]
created: 2026-08-10
updated: 2026-08-10
---

# kubernetes/helm/kubectl provider가 EKS에 인증 못 하던 문제

`Infra/01_infrastructure/providers.tf`의 kubernetes/helm/kubectl provider 설정에서 실제로 겪은 두 가지 문제. 지금은 둘 다 해결돼서 코드에 반영돼 있지만, 왜 지금 형태(exec 방식 + `--role-arn` 명시)인지가 코드 주석에만 짧게 남아있어서 별도 페이지로 정리.

## 증상 1: apply가 오래 걸리면 뒷부분에서 인증 실패

`aws_eks_cluster`+`aws_eks_node_group`처럼 완료까지 시간이 걸리는 리소스들과 `helm_release.argocd`처럼 클러스터 API를 바로 써야 하는 리소스가 같은 apply 안에 섞여 있을 때, apply 앞부분은 성공하는데 뒷부분(클러스터에 실제로 접속해야 하는 리소스)에서 인증이 깨짐.

## 증상 2: `--role-arn` 없이 붙이면 Unauthorized

`aws eks get-token`을 `--role-arn` 없이(= terraform을 실행하는 본인 IAM 신원 그대로) 호출하도록 두면, EKS 쪽에서 `Unauthorized`가 남.

## 원인

**증상 1**: provider 인증 방식을 `data "aws_eks_cluster_auth"` 같은 data source로 **미리 한 번** 토큰을 받아서 고정해두는 방식으로 하면, 그 토큰의 유효기간(~15분)이 apply 전체 소요시간보다 짧을 수 있음 — VPC/EKS 클러스터/노드그룹까지 만드는 apply는 15분을 넘기기 쉬워서, 앞에서 받아둔 토큰이 뒷부분 리소스를 처리할 즈음엔 이미 만료돼 있었음.

**증상 2**: `Infra/modules/eks/access.tf`(→ [[../architecture/terraform-module-boundaries]])에서 EKS Access Entry를 **`cluster_admin` IAM Role한테만** 등록해뒀음(`aws_eks_access_entry.cluster_admin_role`) — 개인 IAM 사용자/역할 각각을 Access Entry에 등록하는 대신, 공유 role 하나만 등록하고 그 role을 assume할 수 있는 사람을 IAM 그룹 멤버십으로 관리하는 구조([[../architecture/terraform-module-boundaries]] 참고). 그래서 terraform을 실행하는 사람의 **원래 신원 그대로는 클러스터에 아무 권한이 없고**, 반드시 `cluster_admin` role을 assume한 신원으로 인증해야 함.

## 해결

두 문제 다 `Infra/01_infrastructure/providers.tf`에서 한 번에 해결됨:

```hcl
provider "kubernetes" {
  host                   = module.eks.cluster_endpoint
  cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority)

  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "aws"
    args        = ["eks", "get-token", "--cluster-name", module.eks.cluster_name, "--region", var.aws_region, "--role-arn", module.eks.cluster_admin_role_arn]
  }
}
```

`helm`/`kubectl` provider도 동일한 `exec` 블록을 각자 갖고 있음(3곳 다 똑같이 맞춰야 함).

- **증상 1 해결**: data source 대신 `exec` 방식 사용 — API를 실제로 호출하는 "그 순간"마다 `aws eks get-token`을 새로 실행해서 매번 새 토큰을 받아옴. apply가 아무리 오래 걸려도 만료된 토큰을 쓸 일이 없음.
- **증상 2 해결**: `exec`의 `args`에 `--role-arn`으로 `module.eks.cluster_admin_role_arn`을 명시 — terraform을 실행하는 사람이 누구든, 토큰 발급 시점에 `cluster_admin` role을 assume한 신원으로 바꿔서 요청하게 함. Access Entry가 이 role한테만 등록돼 있으므로 반드시 이렇게 해야 인증됨.

## 재발 방지

- Terraform에서 EKS(혹은 다른 단명 토큰 기반 API)에 인증하는 provider를 새로 추가할 때는 **data source로 토큰을 미리 고정하지 말 것** — apply 소요시간이 늘어나면 반드시 언젠가 만료 문제를 겪는다. `exec` 방식을 기본으로 쓴다.
- Access Entry를 공유 role 하나에만 등록하는 구조([[../architecture/terraform-module-boundaries]]의 `cluster_admin` 패턴)를 쓰는 한, **kubernetes/helm/kubectl 세 provider 전부 `--role-arn`을 빠짐없이** 넣어야 한다 — 하나라도 빠뜨리면 그 provider가 관리하는 리소스만 골라서 Unauthorized가 남(원인 파악이 헷갈리기 쉬움).

## 관련
- [[../architecture/terraform-module-boundaries]]
- [[github-actions-oidc-not-authorized]] (같은 "EKS/IAM 인증" 계열이지만 원인은 다름 — 이건 GitHub Actions OIDC 쪽 문제)
