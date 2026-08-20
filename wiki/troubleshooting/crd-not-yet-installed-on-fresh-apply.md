---
title: 매일 아침 재적용 시 반복되는 CRD 순서 문제 (ServiceMonitor · SecretStore)
category: troubleshooting
tags: [terraform, kubernetes, crd, kubernetes_manifest, kubectl_manifest, daily-infrastructure-toggle]
created: 2026-08-20
updated: 2026-08-20
---

# 매일 아침 재적용 시 반복되는 CRD 순서 문제

[[../runbook/daily-infrastructure-toggle]]의 "아침 — 켜기" 절차대로 `02_k8s-addon`을 처음부터 다시 apply할 때마다(2026-08-18부터 3일 연속) 정확히 같은 계열의 에러를 겪음. 매번 원인을 다시 설명하는 대신 이번에 제대로 정리.

## 증상 1 — ServiceMonitor

```
Error: API did not recognize GroupVersionKind from manifest (CRD may not be installed)

  with kubernetes_manifest.backend_service_monitor,
  on main.tf line 202, in resource "kubernetes_manifest" "backend_service_monitor":

no matches for kind "ServiceMonitor" in group "monitoring.coreos.com"
```

## 증상 2 — SecretStore

```
Error: argocd/aws-secrets-manager failed to create kubernetes rest client for update of resource:
resource [external-secrets.io/v1/SecretStore] isn't valid for cluster, check the APIVersion and Kind fields are valid

  with kubectl_manifest.argocd_notifications_secret_store,
  on argocd-notifications-eso.tf line 5
```

## 근본 원인 — 같은 패턴, 두 가지 변주

`hashicorp/kubernetes`의 `kubernetes_manifest`(그리고 `gavinbunney/kubectl`의 `kubectl_manifest`도 마찬가지)는 CRD로 정의된 커스텀 리소스를 만들 때 **plan 단계에서 그 CRD가 클러스터에 실제로 있는지 즉시 확인**함. `depends_on`은 apply *순서*만 보장하지, plan이 스키마를 조회하는 시점 자체를 늦춰주지 않음.

- **증상 1(ServiceMonitor)**: 같은 root(`02_k8s-addon`) **안에서** 벌어지는 순서 문제. `module.monitoring`(kube-prometheus-stack)이 이 CRD를 까는데, `kubernetes_manifest.backend_service_monitor`도 같은 root의 같은 apply에 있어서, Terraform이 전체 plan을 짤 때 이 CRD가 아직 없는 상태로 검증을 시도함.
- **증상 2(SecretStore)**: **root 경계를 넘는** 순서 문제라 더 까다로움. `argocd-notifications-eso.tf`(`02_k8s-addon`, ArgoCD 알림 이메일 기능 추가하면서 신설, 커밋 `4e7a965`)가 쓰는 `external-secrets.io/v1/SecretStore` CRD는 **`04_data`의 `module.eso`**가 까는데, 확립된 apply 순서는 `01_infrastructure → 02_k8s-addon → 03_registry/04_data`라서 `02_k8s-addon`이 먼저 돈다. 즉 이 리소스 하나가 실질적으로 그 순서를 거스르는 의존성을 만들어버림.

둘 다 **"매일 밤 `02_k8s-addon`을 통째로 destroy했다가 아침에 처음부터 다시 apply한다"**는 이 프로젝트의 비용 절감 루틴([[../runbook/daily-infrastructure-toggle]]) 때문에 매번 새로 재현됨 — 클러스터를 한 번도 안 지우는 팀이었다면 최초 1회만 겪고 끝났을 문제.

## 해결 (매일 반복해야 하는 절차, 런북에 반영함)

```bash
# 1. 같은 root 안의 CRD 제공자만 먼저
terraform -chdir=02_k8s-addon apply -target=module.monitoring

# 2. 다른 root(04_data)의 CRD 제공자를 순서 거슬러서 먼저
terraform -chdir=04_data workspace select release && terraform -chdir=04_data apply -target=module.eso

# 3. 이제 전체 apply — 두 CRD 다 이미 있으니 정상 통과
terraform -chdir=02_k8s-addon apply
```

## 재발 방지 / 근본적으로 없애려면

이 문서는 "매번 이렇게 우회한다"는 절차만 기록한 것 — 근본 해결책은 다음 중 하나이고 아직 아무것도 적용 안 함:

1. **`argocd-notifications-eso.tf`를 `02_k8s-addon`에서 `04_data`로 옮기기** — ESO(CRD 제공자)와 그걸 쓰는 리소스(SecretStore/ExternalSecret)를 같은 root에 두면 이 root 간 역방향 의존 자체가 사라짐. 다만 ArgoCD(`helm_release.argocd`)는 여전히 `02_k8s-addon`에 있어서, `04_data`에서 `argocd` 네임스페이스의 리소스를 만드는 게 root 책임 경계상 맞는지는 별도 판단 필요
2. **`kubernetes_manifest`/`kubectl_manifest` 대신 CRD를 안 쓰는 방식으로 전환** — 예: ServiceMonitor는 Prometheus Operator 없이 직접 스크랩 설정하는 방법도 있으나 이건 더 큰 구조 변경
3. **그냥 지금처럼 매일 3줄 더 치는 것으로 감수** — 비용 대비 실익을 보면 지금은 이게 제일 현실적. 팀 규모(4인)에서 위 1번 리팩터링에 들일 시간이 있는지가 관건

## 관련
- [[../runbook/daily-infrastructure-toggle]]
- [[../troubleshooting/eks-destroy-layer-separation]]
- [[servicemonitor-actuator-port-mismatch]] — 같은 리소스(`kubernetes_manifest.backend_service_monitor`)에서 겪은 다른 계열 문제(포트 불일치, CRD 자체는 이미 있었던 케이스)
