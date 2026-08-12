---
title: "실제 사고: destroy 순서를 어겨서 생긴 IAM 손상 + ALB/webhook 고아 리소스"
category: troubleshooting
tags: [terraform, eks, iam, alb, eso, webhook, incident]
created: 2026-08-11
updated: 2026-08-11
---

# 실제 사고: destroy 순서를 어겨서 생긴 IAM 손상 + ALB/webhook 고아 리소스

[[eks-destroy-layer-separation]]이 정한 destroy 순서(`04_data` → `02_k8s-addon` → `01_infrastructure`)를 실제로 어겨서 겪은 사고 기록. "이렇게 하면 안 된다"는 걸 설계 문서로만 알고 있는 것과, 실제로 어겼을 때 정확히 뭐가 어떻게 깨지는지는 다른 문제라서 별도로 남긴다.

## 증상

`01_infrastructure`를 `02_k8s-addon`보다 **먼저** destroy하다가, `module.eks`(node group DELETING, `aws_iam_role.cluster_admin`/access-entry/group-policy 삭제)까지 진행된 시점에 `02_k8s-addon destroy`를 시도해서 다음 에러로 완전히 막힘:

```
Kubernetes cluster unreachable: ... exec: executable aws failed with exit code 254
AccessDenied ... sts:AssumeRole on ... team5-qket-cluster-admin
```

`kubectl`/`helm`/`kubectl_manifest` provider가 인증에 쓰는 IAM Role 자체가 이미 지워진 상태라, `02_k8s-addon`이 관리하는 어떤 K8s 리소스에도 손을 댈 수 없는 상황.

## 원인

[[eks-destroy-layer-separation]]의 "증상 1"이 정확히 예고했던 그 문제 — kubernetes/helm provider의 인증 수단(EKS Access Entry용 IAM Role)이 K8s addon(`02_k8s-addon`)이 관리하는 리소스들보다 **먼저** 사라짐. 게다가 그 시점에 EKS node group도 이미 `DELETING`으로 넘어가고 있어서, ALB Controller/ESO 같은 addon 파드 자체도 곧 죽는 이중 위기였음.

## 해결

### 1. IAM 접근 수단부터 복구

destroy가 아니라 **타겟 지정 재-apply**로 방금 지워진 IAM 리소스 4개만 복원:

```bash
terraform apply \
  -target=module.eks.aws_iam_role.cluster_admin \
  -target=module.eks.aws_iam_group_policy.assume_cluster_admin \
  -target=module.eks.aws_eks_access_entry.cluster_admin_role \
  -target=module.eks.aws_eks_access_policy_association.cluster_admin_role
```

node가 완전히 드레인되기 전에 `kubectl` 접근권을 되찾는 게 목적 — 다만 이미 ALB Controller 파드는 `Pending`으로 넘어간 뒤였음(node group 축소가 먼저 시작됐기 때문에 완벽하게 되돌리진 못함).

### 2. 고아가 된 ALB/Target Group/Security Group 수동 정리

ALB Controller 파드가 node 드레인 중에 죽어서 Ingress 삭제를 처리 못 하고 끝남 → AWS 쪽에 ALB/TG/SG가 그대로 남음. AWS CLI로 직접:
- ALB 삭제 (`team5-qket-alb`)
- Target Group 2개 삭제
- Security Group 2개 삭제 — 이 중 하나는 "shared backend" SG라 바로 안 지워지고, 먼저 `eks-cluster-sg-team5-qket-cluster-*`(EKS가 자동 생성하는 클러스터 SG)에 걸린 특정 ingress rule(`sgr-...`, 이 SG를 소스로 지정한 규칙)을 `revoke-security-group-ingress`로 먼저 지운 뒤에야 SG 자체 삭제 가능했음

같은 날 같은 패턴의 사고가 한 번 더 있었음(2번째 발생) — 원인은 동일(ALB Controller가 node 드레인 중 죽어서 Ingress cleanup을 못 함), 대응도 동일.

### 3. `02_k8s-addon destroy`가 Ingress 오브젝트에서 19분+ 멈춤 → `context deadline exceeded`

Ingress 오브젝트가 `finalizers: [group.ingress.k8s.aws/qket]`와 `deletionTimestamp`는 잡혀있는데 영영 안 지워짐. 강제로 finalizer를 지우려 해도:

```
kubectl patch ingress ... --type=merge -p '{"metadata":{"finalizers":[]}}'
# → failed calling webhook "vingress.elbv2.k8s.aws" ...
#   no endpoints available for service "aws-load-balancer-webhook-service"
```

**원인**: ALB Controller 파드는 이미 죽었는데, `ValidatingWebhookConfiguration`/`MutatingWebhookConfiguration`(`aws-load-balancer-webhook`)은 여전히 클러스터에 등록된 채 남아있었음. K8s API 서버는 이 webhook 설정을 보고 "모든 Ingress 변경은 이 webhook 승인을 받아야 한다"고 여전히 믿고 있는데, 정작 그 webhook을 처리할 백엔드(파드)가 없으니 **finalizer 제거 patch 자체가 영원히 거부됨** — 데드락.

**해결**: 그 webhook 설정 자체를 지움(더 이상 유효하지 않은 참조이므로 안전):
```bash
kubectl delete validatingwebhookconfiguration aws-load-balancer-webhook
kubectl delete mutatingwebhookconfiguration aws-load-balancer-webhook
```
그 다음 finalizer patch가 정상적으로 먹히고, namespace terminating이 마무리됨.

**완전히 같은 계열의 문제가 ESO에서도 재발**: `externalsecrets.external-secrets.io` 오브젝트가 `externalsecret-cleanup` finalizer에 걸려 멈췄고, 원인도 동일(ESO 파드도 node 드레인으로 죽음 → `externalsecret-validate`/`secretstore-validate` webhook이 고아로 남음). 같은 방식(webhook 설정 삭제 → finalizer patch)으로 해결.

### 4. IAM Group 삭제 충돌

`01_infrastructure`의 `module.eks` destroy 중 IAM Group `team5-qket-cluster-admins`에서:
```
DeleteConflict: Cannot delete entity, must remove users from group first
```
그룹에 IAM 사용자 5명(`a-student-10`, `a-student-08`, `b-student-03`, `a-student-01`, `b-student-07`)이 여전히 소속돼 있었음 — 이 그룹이 연결하던 Role은 이미 지워져서 멤버십 자체는 이미 의미가 없는 상태였으므로, `aws iam remove-user-from-group`으로 5명 전부 제거 후 `terraform destroy -target=module.eks` 재실행으로 해결.

## 재발 방지

- **destroy 순서(`04_data`/`02_k8s-addon`을 `01_infrastructure`보다 항상 먼저)를 최우선으로 지킨다** — 이 문서는 [[eks-destroy-layer-separation]]이 이미 이론적으로 경고한 문제가 실전에서 어떻게 터지는지 보여주는 실증 사례일 뿐, 근본 대응은 여전히 "순서를 안 어기는 것"이다.
- **admission webhook은 컨트롤러 파드보다 오래 살아남을 수 있다** — 컨트롤러(ALB Controller, ESO 등)가 비정상 종료되면, 그 컨트롤러가 등록해둔 `ValidatingWebhookConfiguration`/`MutatingWebhookConfiguration`이 클러스터에 고아로 남아 **이후 그 리소스 타입에 대한 모든 변경(같은 종류의 Ingress/ExternalSecret 전체)을 막는다**. destroy가 멈추고 원인이 애매한 webhook 에러(`no endpoints available for service ...`)로 나오면 제일 먼저 의심할 지점.
- **`kubectl_manifest`(kubectl provider)는 K8s finalizer를 기다리지 않는다** — `deletionTimestamp`가 찍히는 순간 Terraform 입장에선 "삭제 완료"로 보고 넘어가버려서, 실제 오브젝트가 클러스터에 finalizer와 함께 계속 남아있어도 감지를 못 한다. 이게 애초에 위 Ingress 고아 문제가 커진 이유 중 하나 — 이후 Ingress 리소스는 `kubernetes_ingress_v1`(kubernetes provider 네이티브)로 바꿔서, destroy 시 finalizer가 실제로 clear될 때까지 Terraform이 제대로 기다리게 함(`02_k8s-addon/main.tf`).
- Ingress가 destroy 순서상 ALB Controller보다 먼저 지워지도록 `depends_on = [module.alb_controller]`를 Ingress 쪽에 걸어둠 — Terraform의 `depends_on`은 대상이 **먼저 생성되고 나중에 파괴**되게 만들므로, "ALB Controller가 아직 살아있을 때 Ingress를 먼저 지운다"를 강제하는 방향으로 씀.

## 관련
- [[eks-destroy-layer-separation]]
- [[../runbook/daily-infrastructure-toggle]]
- [[alb-ingressgroup-orphan-on-rename]]
