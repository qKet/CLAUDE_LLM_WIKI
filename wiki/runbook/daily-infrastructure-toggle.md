---
title: 매일 아침/저녁 infrastructure/k8s-addon 켜고 끄기
category: runbook
tags: [infra, terraform, cost]
created: 2026-08-10
updated: 2026-08-10
---

# 매일 아침/저녁 infrastructure/k8s-addon 켜고 끄기

> ⚠️ 처음엔 `Infra/up.sh`/`down.sh` 쉘 스크립트로 만들었다가, "sh 파일은 맥 사용자만 편하게 쓴다(팀원 중 Windows 사용자가 있으면 못 씀)"는 이유로 폐기하고 **`Infra/README.md`에 명령어를 직접 적어두는 방식**으로 바꿨다. 아래 내용은 그 README와 같은 내용 — 실제로 실행할 땐 `Infra/README.md`를 봐도 된다.

## 배경

`01_infrastructure`(EKS/bastion/NAT 등)는 비용 때문에 매일 아침에 켜고 저녁에 끄기로 함. 근데 처음엔 "`01_infrastructure`를 통째로 destroy"를 생각했다가, `04_data`(RDS/Redis, 절대 안 지움)가 `01_infrastructure`의 서브넷 안에 물리적으로 떠있어서 통째로 destroy하면 `04_data`까지 막히거나(DependencyViolation) 위험해질 수 있다는 걸 확인했다.

핵심 깨달음: **VPC/서브넷/라우팅테이블 자체는 떠있어도 과금되지 않는다.** 실제로 돈이 나가는 건 EKS 컨트롤플레인, NAT Gateway, bastion EC2, 노드그룹 EC2뿐. 그리고 [[../architecture/terraform-platform-workload-split]]에서 `04_data`의 rds/redis SG를 bastion SG ID 직접 참조 대신 **서브넷 CIDR 참조**로 바꿔놔서, bastion SG 자체도 매일 지웠다 만들어도 `04_data`엔 전혀 영향이 없다.

그래서 `00_network` 같은 새 root를 또 만드는 대신(그것도 고려했으나 "일이 점점 커진다"는 이유로 보류), **`01_infrastructure` 안에서 `-target`으로 비용 나가는 리소스만 콕 집어 지우는 명령어**로 해결했다.

## 절차

### 저녁 — 끄기

```bash
terraform -chdir=02_k8s-addon destroy

terraform -chdir=01_infrastructure destroy \
  -target=module.eks \
  -target=module.ec2 \
  -target=aws_nat_gateway.this \
  -target=aws_eip.nat \
  -target=module.security_group
```

### 아침 — 켜기

```bash
terraform -chdir=01_infrastructure apply
terraform -chdir=02_k8s-addon apply

# data도 다시 apply — RDS/Redis는 안 건드리고, 네임스페이스가 새로 생기면서
# 같이 사라졌던 ServiceAccount(qket-backend, IRSA)/ConfigMap만 다시 채워짐 (아래 "왜 04_data도" 참고)
terraform -chdir=04_data workspace select release && terraform -chdir=04_data apply
terraform -chdir=04_data workspace select prod && terraform -chdir=04_data apply
```

**지우는 것**: `module.eks`, `module.ec2`(bastion), `aws_nat_gateway.this`, `aws_eip.nat`, `module.security_group`(bastion SG) — 그리고 `02_k8s-addon` 전체(namespace, ArgoCD).

**안 지우는 것**(`01_infrastructure`에 그대로 남음): `module.vpc`, `module.subnet`, 라우팅테이블. `03_registry`(ECR/OIDC)는 애초에 별도 root라 이 과정과 완전히 무관. `04_data`의 AWS 리소스(RDS/Redis/S3)도 이 과정 내내 안 건드려짐 — 단, 아래처럼 K8s 오브젝트 쪽은 예외.

## 왜 아침에 `04_data`도 다시 apply해야 하나

`04_data`는 자기 네임스페이스(`qket-release`/`qket-prod`) **안에** `kubernetes_service_account.backend`(IRSA 연결용)와 `kubernetes_config_map`(DB/Redis/S3 연결 정보) 오브젝트를 만든다. 근데 그 네임스페이스는 `02_k8s-addon`이 관리하는 거라, 저녁에 `02_k8s-addon`을 destroy하면 네임스페이스가 지워지면서 **Kubernetes가 그 안의 모든 오브젝트를 자동으로 같이 지워버린다**(cascade delete) — `04_data`의 Terraform이 직접 뭔가를 지운 게 아닌데도, `04_data`의 state와 실제 K8s 상태가 어긋나게 된다(drift).

다행히 `terraform apply`는 refresh를 먼저 하기 때문에, RDS/Redis(진짜 AWS 리소스)는 그대로 두고 **사라진 K8s 오브젝트 3개(ServiceAccount 1 + ConfigMap 2)만 딱 찾아서 다시 만들어준다** — 비용도 리스크도 없는 몇 초짜리 작업이라, 그냥 아침 루틴에 세 번째 단계로 넣으면 된다.

> 이걸 근본적으로 없애려면(K8s 오브젝트가 아니라 AWS Secrets Manager처럼 EKS와 무관하게 살아있는 곳에 연결 정보를 두는 것) `backup/modules/eso`(External Secrets Operator, 보류 중)를 재활성화해야 하는데, 지금 당장 할 만한 작업은 아니라서 일단 "아침에 한 번 더 apply"로 대응하기로 함.

## 순서가 중요한 이유

`02_k8s-addon`(ArgoCD/namespace)을 먼저 완전히 지워야 한다 — EKS Access Entry가 아직 살아있는 상태에서 지워야 kubernetes/helm provider 인증이 정상 동작하기 때문. 이 순서를 안 지키면 [[../troubleshooting/eks-provider-auth]]/[[../troubleshooting/eks-destroy-layer-separation]]에서 겪었던 `Unauthorized`가 재현된다. 켤 때는 반대로 `01_infrastructure`가 완전히 떠서 EKS가 준비된 뒤에 `02_k8s-addon`을, 그 다음 `04_data`를 apply해야 한다(네임스페이스가 있어야 `04_data`의 K8s 오브젝트를 만들 수 있으므로).

## 주의사항

- `-target`으로 NAT Gateway를 지우면 `aws_route_table.private`(일반 워크로드용)의 라우트 하나가 사라지는데, 이건 정상이다 — 아침에 NAT를 다시 만들면서 라우트도 자동으로 복구된다. `aws_route_table.private_data`(RDS/Redis용)는 애초에 인터넷 라우트가 없는 테이블이라 이 과정과 무관.
- `01_infrastructure`/`02_k8s-addon`/`04_data`가 이미 최소 한 번 apply된 상태를 전제로 한다 — [[terraform-apply-order]](최초 적용 절차)를 먼저 따를 것.
- Helm 차트로 backend를 배포한다면, 그 차트가 자기 ServiceAccount를 새로 만들지 않고(`serviceAccount.create: false`) `qket-backend`(Terraform이 만든 것)를 참조하도록 되어 있는지 확인 — 안 그러면 IRSA가 안 붙어서 파드가 못 뜬다.
- `-chdir` 대신 `cd 01_infrastructure && terraform ...`처럼 각자 편한 방식으로 실행해도 된다 — `-chdir`은 셸 종류(bash/zsh/PowerShell 등)를 안 타는 `terraform` 자체 플래그라서 이렇게 적어둠.

## 관련
- [[terraform-apply-order]]
- [[../architecture/terraform-platform-workload-split]]
- [[../troubleshooting/eks-destroy-layer-separation]]
- [[../troubleshooting/eks-provider-auth]]
