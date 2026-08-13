---
title: KEDA ScaledObject(CPU 트리거)가 부하를 줘도 전혀 스케일 안 함 — 원인은 metrics-server 부재
category: troubleshooting
tags: [monitoring, keda, hpa, metrics-server, autoscaling, kubernetes, load-test]
created: 2026-08-13
updated: 2026-08-13
---

# KEDA ScaledObject(CPU 트리거)가 부하를 줘도 전혀 스케일 안 함 — 원인은 metrics-server 부재

## 증상

[[../architecture/keda-autoscaling|KEDA 도입]] 후 첫 k6 부하테스트(1000명 동시 로그인)를 돌렸는데, 부하가 명백히 걸리고 있는데도(응답 지연, 이후 확인된 CPU 스로틀링) `qket-backend` replica가 4개에서 단 한 번도 안 늘어남.

```
kubectl get hpa -n qket-release
NAME                                REFERENCE                 TARGETS            MINPODS   MAXPODS   REPLICAS
keda-hpa-qket-backend-scaledobject  Deployment/qket-backend   cpu: <unknown>/70%  4         8         4
```

`TARGETS`가 실제 숫자가 아니라 `<unknown>`으로 영구히 멈춰있는 게 이상 징후.

## 원인 진단

`kubectl describe hpa keda-hpa-qket-backend-scaledobject -n qket-release`로 Condition을 직접 확인:

```
Warning  FailedGetResourceMetric  ... unable to fetch metrics from resource metrics API:
the server could not find the requested resource (get pods.metrics.k8s.io)
```

`pods.metrics.k8s.io` API 자체가 클러스터에 없다는 뜻 — 이건 KEDA/HPA 설정 문제가 아니라, HPA/KEDA(cpu 트리거)가 공통으로 의존하는 **`metrics-server`가 클러스터에 아예 설치돼 있지 않았던 것**. 돌이켜보면 그 전에 `kubectl top pods`를 시도했을 때도 비슷하게 실패했었는데, 그때는 원인을 깊이 안 팠었음.

`metrics-server`는 EKS 관리형 애드온이 아니라서 자동으로 안 깔림 — 명시적으로 설치해야 함.

## 해결

`modules/addons/metrics-server` 신설, `02_k8s-addon`에 연결:

```hcl
resource "helm_release" "metrics_server" {
  name       = "metrics-server"
  repository = "https://kubernetes-sigs.github.io/metrics-server/"
  chart      = "metrics-server"
  namespace  = "kube-system"

  set {
    name  = "args[0]"
    value = "--kubelet-insecure-tls"
  }
}
```

`--kubelet-insecure-tls`가 필요한 이유: EKS 워커 노드의 kubelet serving 인증서가 metrics-server가 기본 신뢰하는 CA로 서명돼있지 않은 경우가 많음(EKS 기본 구성의 잘 알려진 제약) — 이거 없으면 스크래핑 자체가 `x509: cannot validate certificate`로 실패함. 클러스터 내부 통신이라 노출 위험은 없음.

## 검증

설치 후 재테스트: `kubectl get hpa`가 `cpu: 1%/70%`처럼 실제 숫자를 보여주기 시작했고, 이어진 부하테스트에서 replica가 4 → 8까지 실제로 늘어나는 걸 확인함.

## 교훈

HPA/KEDA는 `metrics-server`가 없어도 리소스 자체는 정상 생성되고, 겉으로 보기엔 "그냥 안 늘어나는" 것처럼만 보임 — 에러가 사용자에게 전혀 노출되지 않고 `kubectl describe hpa`의 Condition까지 들어가야만 원인이 보임. 오토스케일링 도입 시 `metrics-server` 설치를 체크리스트 1번으로 둘 것.

## 관련
- [[../architecture/keda-autoscaling]]
- [[backend-cpu-throttling-and-scaling-load-test]]
