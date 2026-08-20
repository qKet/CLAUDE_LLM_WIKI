---
title: 매일 아침 재적용 시 반복되는 CRD 순서 문제 (ServiceMonitor · SecretStore)
category: troubleshooting
tags: [terraform, kubernetes, crd, kubernetes_manifest, kubectl_manifest, daily-infrastructure-toggle]
created: 2026-08-20
updated: 2026-08-20
---

# 매일 아침 재적용 시 반복되는 CRD 순서 문제

> ✅ 2026-08-20: 증상 1(ServiceMonitor)은 근본 해결됨 — `kubernetes_manifest.backend_service_monitor`를 `helm_release` 기반 모듈(`modules/addons/backend-servicemonitor`)로 전환해서, CRD와 그걸 쓰는 오브젝트를 아예 다른 방식(plan-time 스키마 검증이 없는 리소스 타입)으로 만들게 바꿈. 아래 "해결" 절차 중 `module.monitoring` 선적용 스텝은 더 이상 필요 없음. 증상 2(SecretStore)도 같은 방식으로 `kubectl_manifest`→`helm_release`(`modules/addons/argocd-notifications-secrets`)로 전환했지만, ESO가 **다른 root(04_data)** 에 있는 cross-root 문제라 `module.eso` 선적용은 여전히 필요함 — 이 전환으로 없어진 건 "ESO가 아직 없으면 02_k8s-addon 전체 plan/apply가 통째로 막히던 것"뿐이고, 순서 자체(런북 절차)는 그대로 유지됨. 설계 근거는 [[../decisions/2026-08-20-ingress-to-gateway-api-migration]] 참고.

[[../runbook/daily-infrastructure-toggle]]의 "아침 — 켜기" 절차대로 `02_k8s-addon`을 처음부터 다시 apply할 때마다(2026-08-18부터 3일 연속) 정확히 같은 계열의 에러를 겪음. 매번 원인을 다시 설명하는 대신 이번에 제대로 정리. (아래 내용은 2026-08-20 이전, 최초 발견 당시 기록 — 위 갱신 노트와 함께 읽을 것)

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

> ✅ 2026-08-20: 아래 옵션 중 하나(CRD를 쓰는 리소스 타입 자체를 바꾸기)를 실제로 적용해서 증상 1(ServiceMonitor)을 완전히 없앴고, 증상 2(SecretStore)도 "plan 단계에서 전체가 막히는 것"까지는 없앴다. 자세한 구현/트레이드오프는 [[../decisions/2026-08-20-ingress-to-gateway-api-migration]] 참고 — 이 문서엔 요약만 남김.

1. ✅ **적용됨 — `kubernetes_manifest`/`kubectl_manifest` 대신 `helm_release`로 전환**: CRD를 안 쓰는 게 아니라, CRD를 **쓰는 방식**을 바꿨다. `kubernetes_manifest`/`kubectl_manifest`는 plan 단계에서 대상 kind의 스키마(CRD)가 이미 있는지 API 서버에 확인하는데, `helm_release`는 테라폼이 "이 차트를 설치해라"는 선언만 diff할 뿐 차트 내용물의 kind를 전혀 확인 안 한다. 그래서 CRD를 쓰는 오브젝트(ServiceMonitor, SecretStore/ExternalSecret, Gateway API의 GatewayClass/Gateway/HTTPRoute)를 로컬 Helm 차트의 `templates/`로 감싸서 `helm_release`로 설치하면 이 plan-time 문제 자체가 없어진다.
   - **같은 root 안에서 CRD 제공자와 소비자가 만나는 경우(ServiceMonitor)**: `depends_on`만 걸면 완전히 해결됨 — `modules/addons/backend-servicemonitor`가 그 예. 매일 아침 `-target` 선적용이 통째로 필요 없어짐.
   - **CRD 제공자가 "다른 root"에 있는 경우(SecretStore — ESO는 04_data)**: 이 전환으로도 cross-root 순서 문제 자체는 못 없앤다(Helm의 crds/→templates/ 순서 보장은 "같은 차트/같은 apply" 안에서만 유효). 대신 실패 범위가 "전체 plan/apply가 막힘"에서 "이 helm_release 하나만 apply 시점에 실패"로 좁아짐 — `04_data`의 `module.eso` 선적용은 여전히 필요함(`modules/addons/argocd-notifications-secrets`가 그 예).
2. **`argocd-notifications-eso.tf`(현 `modules/addons/argocd-notifications-secrets`)를 `02_k8s-addon`에서 `04_data`로 옮기기** — ESO(CRD 제공자)와 그걸 쓰는 리소스를 같은 root에 두면 cross-root 순서 문제 자체가 사라짐(위 1번의 "helm_release 전환"과 같이 적용하면 `04_data` 선적용 스텝까지 완전히 없앨 수 있음). 다만 ArgoCD(`helm_release.argocd`)는 여전히 `02_k8s-addon`에 있어서, `04_data`에서 `argocd` 네임스페이스의 리소스를 만드는 게 root 책임 경계상 맞는지는 별도 판단 필요 — **미적용**
3. **그냥 지금처럼 매일 몇 줄 더 치는 것으로 감수** — SecretStore 쪽에 여전히 남아있는 `module.eso` 선적용 한 줄에 한해서는 이 방식 유지 중

## 관련
- [[../runbook/daily-infrastructure-toggle]]
- [[../troubleshooting/eks-destroy-layer-separation]]
- [[servicemonitor-actuator-port-mismatch]] — 같은 리소스(`kubernetes_manifest.backend_service_monitor`)에서 겪은 다른 계열 문제(포트 불일치, CRD 자체는 이미 있었던 케이스)
