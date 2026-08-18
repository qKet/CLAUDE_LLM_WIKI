---
title: Cluster Autoscaler 구조 — 노드 레벨 오토스케일링
category: architecture
tags: [cluster-autoscaler, eks, autoscaling, node-group, irsa]
created: 2026-08-18
updated: 2026-08-18
---

# Cluster Autoscaler 구조 — 노드 레벨 오토스케일링

## 구조 / 요약

EKS 관리형 노드그룹(`modules/eks`의 `aws_eks_node_group`)의 `desired_size`를 `min_size`(1)~`max_size`(3, `01_infrastructure/variables.tf`, t3.xlarge) 사이에서 자동으로 조절해주는 컨트롤러.

- **설치 위치**: `Infra/modules/addons/cluster-autoscaler` → `02_k8s-addon`에서 `helm_release`로 설치(다른 addon들과 같은 패턴, IRSA + helm chart)
- **차트**: `https://kubernetes.github.io/autoscaler`의 `cluster-autoscaler` — `image.tag`를 `v{eks_version}.0`으로 클러스터 쿠버네티스 버전에 맞춰 고정(`eks_version`은 `01_infrastructure`가 output으로 노출, 클러스터 버전 올릴 때 같이 올려야 함)
- **autodiscovery**: EKS 관리형 노드그룹은 밑단 ASG에 `k8s.io/cluster-autoscaler/enabled=true`, `k8s.io/cluster-autoscaler/<클러스터명>=owned` 태그를 **AWS가 자동으로 붙여줌** — 확인됨(2026-08-18). 그래서 ASG 쪽 Terraform은 전혀 안 건드리고 helm 설치만으로 바로 그 태그를 찾아서 동작함
- **IAM 권한 스코프**: 읽기(`Describe*`)는 전체 허용, 실제로 노드 수를 바꾸는 쓰기(`SetDesiredCapacity`/`TerminateInstanceInAutoScalingGroup`/`UpdateAutoScalingGroup`)는 `aws:ResourceTag/k8s.io/cluster-autoscaler/<클러스터명>=owned` 조건으로 제한 — 같은 AWS 계정에 다른 팀 클러스터(`team1-eks`, `doro-erp-dev`)도 있어서, 실수로 남의 ASG를 건드리지 못하게 스코프를 걸어둠

## KEDA와의 관계 — 레이어가 다름

[[keda-autoscaling]](파드 오토스케일링)과 이 문서(노드 오토스케일링)는 서로 다른 레이어를 담당함:

- KEDA는 `qket-backend`/`qket-frontend` 파드 수를 CPU 사용률 기준으로 4~8개 사이에서 조절
- 그 파드들을 실제로 얹을 노드가 부족하면(2노드로는 baseline만으로도 CPU 60%대) KEDA가 아무리 replica를 늘려도 새 파드는 `Pending`으로 멈춤
- Cluster Autoscaler가 그 밑단(노드 수)을 받쳐줘야 KEDA의 `maxReplicas`가 실제로 의미를 가짐

2026-08-18 [[../decisions/2026-08-18-capacity-planning-large-traffic-readiness]] 분석에서 이 컨트롤러 자체가 없어서 `node_max_size=3`이 사실상 장식값이었다는 게 드러나서 설치하게 됨.

## Karpenter 대신 이걸 선택한 이유

- 노드 인스턴스 타입이 1종(t3.xlarge), 노드 수도 최대 3대뿐 — Karpenter의 강점(다양한 인스턴스 조합, Spot 최적화, 빠른 빈패킹)이 발휘될 여지가 적음
- 반대로 Karpenter는 기존 "고정 노드그룹 1개" 모델 자체를 재설계해야 해서(NodePool/EC2NodeClass CRD, 인터럽션 큐 등) 지금 규모엔 비용 대비 효과가 안 맞음
- Cluster Autoscaler는 이미 Terraform에 선언돼 있던 min/max를 그대로 활용하는 가장 낮은 비용의 선택 — ASG 태그도 이미 자동으로 붙어있어서 노드그룹 코드를 한 줄도 안 건드림

## 의도적으로 어색하거나 특이한 지점

- **최초 apply 직후 파드가 1번 크래시하는 게 정상** — `AccessDenied: sts:AssumeRoleWithWebIdentity`로 죽었다가 Kubernetes가 자동 재시작하면 바로 정상화됨. IAM Role/ServiceAccount가 막 생성된 직후라 IRSA(OIDC federated AssumeRoleWithWebIdentity) trust policy 전파에 살짝 지연이 있어서 생기는 레이스 — ALB Controller의 admission webhook 레이스([[../troubleshooting/eks-destroy-layer-separation]])와 같은 계열. `02_k8s-addon`을 매일 아침 재적용하는 루틴([[../runbook/daily-infrastructure-toggle]])에서 매번 재현될 수 있으나 재시작 횟수가 1에서 안 늘어나고 `Running`이면 정상이니 조치 불필요
- **`extraArgs.skip-nodes-with-system-pods=false`를 명시적으로 꺼둠** — 기본값(true)이면 ALB Controller/metrics-server/cluster-autoscaler 자기 자신처럼 `kube-system`에 떠있는 DaemonSet 아닌 파드가 하나라도 있는 노드는 스케일다운 후보에서 제외돼서, 이 프로젝트처럼 addon을 `kube-system`에 두는 구성에서는 사실상 노드가 영원히 안 줄어드는 함정이 됨
- **노드그룹은 여전히 인스턴스 타입 1종(t3.xlarge)** — Cluster Autoscaler는 이 노드그룹의 desired_size만 조절할 뿐, 인스턴스 타입 다양화나 Spot 활용은 안 함(그건 Karpenter의 영역)

## 관련
- [[keda-autoscaling]]
- [[../decisions/2026-08-18-capacity-planning-large-traffic-readiness]]
- [[../troubleshooting/eks-destroy-layer-separation]]
