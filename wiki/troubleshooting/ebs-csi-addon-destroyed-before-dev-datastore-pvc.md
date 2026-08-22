---
title: 컨트롤러가 자기가 관리하는 CR/PV보다 먼저 destroy돼서 finalizer가 영원히 안 풀리는 문제 (EBS CSI, KEDA)
category: troubleshooting
tags: [terraform, ebs-csi, keda, dev-datastore, destroy, finalizer, namespace]
created: 2026-08-22
updated: 2026-08-22
---

> 🟢 2026-08-22 같은 날 후속 — **완전히 같은 클래스의 문제가 KEDA에서도 재현됨.** 이번엔
> `kubernetes_namespace.qket`(release/prod) 자체가 `Terminating`에서 안 넘어감 — 원인은
> `qket-backend-scaledobject`/`qket-frontend-scaledobject`(ArgoCD/CD Helm 차트가 만든
> ScaledObject, 네임스페이스당 2개씩 총 4개)에 걸린 `finalizer.keda.sh`가 안 풀린 것.
> `keda` 네임스페이스에 파드가 하나도 없고 `helm list -A`에도 keda 릴리즈 자체가 없었음 —
> `module.keda`가 네임스페이스보다 먼저 destroy돼버려서 그 finalizer를 처리해줄 KEDA
> 오퍼레이터가 이미 사라진 상태였음. 4개 ScaledObject 전부 finalizer 강제 제거로 해결.
> 근본 수정: `kubernetes_namespace.qket`에 `depends_on = [module.keda, module.eso_controller]`
> 추가(`module.eso_controller`도 같은 이유로 선제적으로 같이 묶어둠 — 04_data의 module.eso가
> 이 네임스페이스에 SecretStore/ExternalSecret을 만드는데 그 컨트롤러도 이 시나리오에 취약함).
> 아래 원본 내용(EBS CSI 사례)은 그대로 두고, 이 문제가 "특정 addon 하나"가 아니라 **"자기가
> 관리하는 리소스를 담은 네임스페이스/PV보다 컨트롤러 자신이 먼저 destroy될 수 있는 모든
> addon"에서 재현 가능한 일반적인 패턴**이라는 게 이번에 확인됨.

# 컨트롤러가 자기가 관리하는 CR/PV보다 먼저 destroy돼서 finalizer가 영원히 안 풀리는 문제

## 사례 1: EBS CSI addon이 dev-datastore PVC/PV보다 먼저 destroy됨

### 증상

매일 밤 `02_k8s-addon` destroy 중 `module.dev_datastore.kubernetes_persistent_volume_claim.mysql`,
`kubernetes_persistent_volume.redis`가 `Still destroying...`를 **몇 분이고 계속 반복**(15분 넘게
진행되어도 안 끝남) — 겉보기엔 무한루프처럼 보임.

### 원인

`aws_eks_addon.ebs_csi`(EBS CSI 드라이버)와 `module.dev_datastore`(PV/PVC/StatefulSet) 사이에
Terraform이 아는 의존 관계가 전혀 없었음 — `dev_datastore`의 PV가 CSI 드라이버를
`"ebs.csi.aws.com"`이라는 **문자열**로만 참조해서, Terraform 그래프 입장에서는 둘이 완전히
무관한 리소스였음. 그래서 destroy 순서가 우연에 맡겨져 있었는데, 하필 이날은 `ebs_csi` addon이
`dev_datastore`의 PVC/PV보다 **먼저** 지워짐.

`aws eks list-addons`로 확인해보니 addon 목록이 완전히 비어있었음 — EBS CSI 컨트롤러(외부
attacher 포함) 파드 자체가 이미 사라진 상태. PVC(`kubernetes.io/pvc-protection`)/PV
(`external-attacher/ebs-csi-aws-com`) finalizer는 그 컨트롤러가 "이 볼륨 detach 처리 끝났다"고
확인해줘야 풀리는데, 그 컨트롤러가 이번 destroy 안에서 다시 살아날 일이 없으니 **영원히 안 풀림.**

### 해결

당장 막힌 것 풀기 — PVC/PV/VolumeAttachment의 finalizer를 강제로 제거:
```bash
kubectl patch pvc dev-mysql-data -n qket-release -p '{"metadata":{"finalizers":[]}}' --type=merge
kubectl patch pv dev-mysql-pv -p '{"metadata":{"finalizers":[]}}' --type=merge
kubectl patch pv dev-redis-pv -p '{"metadata":{"finalizers":[]}}' --type=merge
# 아직 남아있는 VolumeAttachment도 동일하게(또는 deletionTimestamp가 없으면 그냥 delete)
kubectl patch volumeattachment <name> -p '{"metadata":{"finalizers":[]}}' --type=merge
```
안전한 이유: `modules/addons/dev-datastore`의 PV는 `persistent_volume_reclaim_policy = "Retain"`
(EBS 볼륨 자체는 `03_registry`가 영구 소유) — 이 K8s 오브젝트(PVC/PV/VolumeAttachment)를 강제로
지워도 실제 EBS 볼륨 데이터는 전혀 안 건드림. finalizer 제거 후 `terraform destroy`가 다음
폴링에서 오브젝트가 사라진 걸 확인하고 자동으로 이어서 진행됨.

근본 원인 수정 — `02_k8s-addon/main.tf`의 `module.dev_datastore`에
`depends_on = [..., aws_eks_addon.ebs_csi]` 추가. Terraform은 `depends_on`을 destroy 시
역순으로 적용하므로:
- **create**: `ebs_csi` 먼저 준비된 뒤 `dev_datastore`(PV/파드) 생성 — 오히려 더 안전해짐
- **destroy**: `dev_datastore` 먼저 지워진 뒤 `ebs_csi` 제거 — 오늘 겪은 문제가 재발 안 함

## 사례 2: KEDA가 네임스페이스보다 먼저 destroy됨 (같은 날 후속)

### 증상

`kubernetes_namespace.qket["release"]`/`["prod"]`가 `Still destroying...`를 몇 분째 반복 —
`kubectl get namespace qket-release -o json`의 `status.conditions`로 정확한 원인을 바로 특정 가능:
```
NamespaceContentRemaining: Some resources are remaining: scaledobjects.keda.sh has 2 resource instances
NamespaceFinalizersRemaining: Some content in the namespace has finalizers remaining: finalizer.keda.sh in 2 resource instances
```
(release/prod 각각 backend/frontend ScaledObject 2개씩, 총 4개)

### 원인

`module.keda`(KEDA 오퍼레이터, Terraform 관리)와 `kubernetes_namespace.qket` 사이에도 사례 1과
똑같이 Terraform이 아는 의존 관계가 없었음 — ScaledObject 자체는 Terraform이 아니라 **ArgoCD가
CD Helm 차트로 만든 리소스**라 더더욱 그 관계가 안 보임. `module.keda`가 네임스페이스보다 먼저
destroy되면서 KEDA 오퍼레이터(`finalizer.keda.sh`를 처리해줄 유일한 주체)가 이미 사라져서
ScaledObject들이 영원히 안 지워지고, 그래서 네임스페이스 자체도 못 지워짐.
`helm list -A`에 keda 릴리즈가 아예 없고 `keda` 네임스페이스에 파드가 0개인 것으로 확인.

### 해결

```bash
kubectl patch scaledobject qket-backend-scaledobject -n qket-prod -p '{"metadata":{"finalizers":[]}}' --type=merge
kubectl patch scaledobject qket-frontend-scaledobject -n qket-prod -p '{"metadata":{"finalizers":[]}}' --type=merge
kubectl patch scaledobject qket-backend-scaledobject -n qket-release -p '{"metadata":{"finalizers":[]}}' --type=merge
kubectl patch scaledobject qket-frontend-scaledobject -n qket-release -p '{"metadata":{"finalizers":[]}}' --type=merge
```
(같은 destroy 안에서 `qket-release`에 leftover 파드 `dev-mysql-0`도 하나 더 걸려있었음 —
`kubectl delete pod dev-mysql-0 -n qket-release --grace-period=0 --force`로 같이 정리. StatefulSet은
이미 삭제됐는데 파드 객체만 종료 처리가 안 끝나고 남아있던 경우.)

근본 원인 수정 — `kubernetes_namespace.qket`에 `depends_on = [module.keda, module.eso_controller]`
추가. `module.eso_controller`도 선제적으로 같이 묶어둠 — 04_data의 `module.eso`가 이
네임스페이스에 SecretStore/ExternalSecret(`external-secrets.io`) CR을 만드는데, 그 컨트롤러도
정확히 같은 취약점을 갖고 있어서 언젠가 재현될 수 있다고 판단.

## 재발 방지 (일반화)

이 문제는 "EBS CSI"나 "KEDA" 하나만의 특수 사정이 아니라, **다음 조건을 모두 만족하는 모든
addon에서 재현 가능한 일반적인 패턴**임:
1. 어떤 컨트롤러(addon)가 관리하는 CR/PV에 그 컨트롤러 전용 finalizer가 걸려있고
2. 그 CR/PV가 Terraform이 아닌 다른 경로(ArgoCD, 다른 root)로 만들어질 수도 있어서
3. Terraform 그래프상 그 컨트롤러와 CR/PV를 담은 리소스(네임스페이스, PV 등) 사이에
   명시적 의존 관계가 없는 경우

→ 새 addon을 추가하거나 기존 addon이 이런 CR을 만든다면, 그 CR/PV를 담은 리소스(주로
`kubernetes_namespace.qket`)에 `depends_on`으로 그 컨트롤러를 명시적으로 묶어둘 것 — CRD 있는
addon(cert-manager, external-dns 등 나중에 뭐가 추가되든)을 새로 넣을 때마다 이 체크리스트를
확인. 이번 것과 비슷한 유형의 순서 문제는 [[destroy-order-incident-and-webhook-orphans]],
[[eks-destroy-layer-separation]]에도 이미 기록돼 있음 — 이번 건들은 "서로 다른 root 사이"가
아니라 "같은 root(02_k8s-addon) 안에서" 생긴 버전.

## 관련
- [[destroy-order-incident-and-webhook-orphans]]
- [[eks-destroy-layer-separation]]
- [[../runbook/daily-infrastructure-toggle]]
