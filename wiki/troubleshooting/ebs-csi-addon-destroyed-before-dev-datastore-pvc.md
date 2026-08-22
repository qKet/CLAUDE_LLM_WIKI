---
title: EBS CSI addon이 dev-datastore PVC/PV보다 먼저 destroy돼서 terraform destroy가 무한 대기
category: troubleshooting
tags: [terraform, ebs-csi, dev-datastore, destroy, finalizer]
created: 2026-08-22
updated: 2026-08-22
---

# EBS CSI addon이 dev-datastore PVC/PV보다 먼저 destroy돼서 `terraform destroy`가 무한 대기

## 증상

매일 밤 `02_k8s-addon` destroy 중 `module.dev_datastore.kubernetes_persistent_volume_claim.mysql`,
`kubernetes_persistent_volume.redis`가 `Still destroying...`를 **몇 분이고 계속 반복**(15분 넘게
진행되어도 안 끝남) — 겉보기엔 무한루프처럼 보임.

## 원인

`aws_eks_addon.ebs_csi`(EBS CSI 드라이버)와 `module.dev_datastore`(PV/PVC/StatefulSet) 사이에
Terraform이 아는 의존 관계가 전혀 없었음 — `dev_datastore`의 PV가 CSI 드라이버를
`"ebs.csi.aws.com"`이라는 **문자열**로만 참조해서, Terraform 그래프 입장에서는 둘이 완전히
무관한 리소스였음. 그래서 destroy 순서가 우연에 맡겨져 있었는데, 하필 이날은 `ebs_csi` addon이
`dev_datastore`의 PVC/PV보다 **먼저** 지워짐.

`aws eks list-addons`로 확인해보니 addon 목록이 완전히 비어있었음 — EBS CSI 컨트롤러(외부
attacher 포함) 파드 자체가 이미 사라진 상태. PVC(`kubernetes.io/pvc-protection`)/PV
(`external-attacher/ebs-csi-aws-com`) finalizer는 그 컨트롤러가 "이 볼륨 detach 처리 끝났다"고
확인해줘야 풀리는데, 그 컨트롤러가 이번 destroy 안에서 다시 살아날 일이 없으니 **영원히 안 풀림.**

## 해결

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

## 재발 방지

- 앞으로 어떤 모듈이든 "K8s 매니페스트 안에서 문자열로만 다른 addon을 참조"하는 패턴(CSI 드라이버 이름, IngressClass 이름 등)이 있으면, Terraform 그래프에는 그 관계가 안 보인다는 걸 기억할 것 — 필요하면 `depends_on`으로 명시적으로 걸어야 함. 이번 것과 비슷한 유형의 순서 문제는 [[destroy-order-incident-and-webhook-orphans]], [[eks-destroy-layer-separation]]에도 이미 기록돼 있음 — 이번 건은 "서로 다른 root 사이"가 아니라 "같은 root(02_k8s-addon) 안에서" 생긴 버전.

## 관련
- [[destroy-order-incident-and-webhook-orphans]]
- [[eks-destroy-layer-separation]]
- [[../runbook/daily-infrastructure-toggle]]
