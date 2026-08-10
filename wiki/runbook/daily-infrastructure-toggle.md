---
title: 매일 아침/저녁 infrastructure 켜고 끄기 (up.sh / down.sh)
category: runbook
tags: [infra, terraform, cost]
created: 2026-08-10
updated: 2026-08-10
---

# 매일 아침/저녁 infrastructure 켜고 끄기

## 배경

`01_infrastructure`(EKS/bastion/NAT 등)는 비용 때문에 매일 아침에 켜고 저녁에 끄기로 함. 근데 처음엔 "`01_infrastructure`를 통째로 destroy"를 생각했다가, `04_data`(RDS/Redis, 절대 안 지움)가 `01_infrastructure`의 서브넷 안에 물리적으로 떠있어서 통째로 destroy하면 `04_data`까지 막히거나(DependencyViolation) 위험해질 수 있다는 걸 확인했다.

핵심 깨달음: **VPC/서브넷/라우팅테이블 자체는 떠있어도 과금되지 않는다.** 실제로 돈이 나가는 건 EKS 컨트롤플레인, NAT Gateway, bastion EC2, 노드그룹 EC2뿐. 그리고 [[../architecture/terraform-platform-workload-split]]에서 `04_data`의 rds/redis SG를 bastion SG ID 직접 참조 대신 **서브넷 CIDR 참조**로 바꿔놔서, bastion SG 자체도 매일 지웠다 만들어도 `04_data`엔 전혀 영향이 없다.

그래서 `00_network` 같은 새 root를 또 만드는 대신(그것도 고려했으나 "일이 점점 커진다"는 이유로 보류), **`01_infrastructure` 안에서 `-target`으로 비용 나가는 리소스만 콕 집어 지우는 스크립트**로 해결했다.

## 절차

```bash
cd Infra
./down.sh   # 저녁 — 02_k8s-addon 전체 destroy → 01_infrastructure의 EKS/bastion/NAT만 targeted destroy
./up.sh     # 아침 — 01_infrastructure apply → 02_k8s-addon apply (지워진 것만 재생성)
```

`down.sh`가 지우는 것: `module.eks`, `module.ec2`(bastion), `aws_nat_gateway.this`, `aws_eip.nat`, `module.security_group`(bastion SG) — 그리고 `02_k8s-addon` 전체(namespace, ArgoCD).

`down.sh`가 **안 지우는 것**(`01_infrastructure`에 그대로 남음): `module.vpc`, `module.subnet`, 라우팅테이블, `module.ecr`, `module.github_actions_oidc`. `04_data`는 이 스크립트가 도는 동안 한 번도 안 건드려짐.

## 순서가 중요한 이유

`02_k8s-addon`(ArgoCD/namespace)을 먼저 완전히 지워야 한다 — EKS Access Entry가 아직 살아있는 상태에서 지워야 kubernetes/helm provider 인증이 정상 동작하기 때문. 이 순서를 안 지키면 [[../troubleshooting/eks-provider-auth]]/[[../troubleshooting/eks-destroy-layer-separation]]에서 겪었던 `Unauthorized`가 재현된다. `up.sh`는 반대로 `01_infrastructure`가 완전히 떠서 EKS가 준비된 뒤에 `02_k8s-addon`을 apply해야 한다.

## 주의사항

- `-target`으로 NAT Gateway를 지우면 `aws_route_table.private`(일반 워크로드용)의 라우트 하나가 사라지는데, 이건 정상이다 — `up.sh`가 NAT를 다시 만들면서 라우트도 자동으로 복구된다. `aws_route_table.private_data`(RDS/Redis용)는 애초에 인터넷 라우트가 없는 테이블이라 이 과정과 무관.
- `03_registry`(구상 단계, ECR/OIDC를 여기서 분리하는 것)가 나중에 실제로 만들어지면, `module.ecr`/`module.github_actions_oidc`가 `01_infrastructure`에서 빠지므로 이 스크립트의 `-target` 목록은 안 바뀌지만(원래도 안 건드리던 것들이라) 배경 설명은 갱신할 것.
- 이 스크립트는 `01_infrastructure`/`02_k8s-addon`이 이미 최소 한 번 apply된 상태를 전제로 한다 — [[terraform-apply-order]](최초 적용 절차)를 먼저 따를 것.

## 관련
- [[terraform-apply-order]]
- [[../architecture/terraform-platform-workload-split]]
- [[../troubleshooting/eks-destroy-layer-separation]]
- [[../troubleshooting/eks-provider-auth]]
