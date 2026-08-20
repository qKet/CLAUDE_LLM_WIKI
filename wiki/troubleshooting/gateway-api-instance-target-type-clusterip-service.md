---
title: Gateway API TargetGroupConfiguration targetType 기본값(instance)이 ClusterIP 서비스에서 Gateway 전체를 Accepted:False로 실패시킴
category: troubleshooting
tags: [gateway-api, alb, target-group, aws-load-balancer-controller]
created: 2026-08-20
updated: 2026-08-20
---

# TargetGroupConfiguration targetType 기본값이 ClusterIP 서비스에서 Gateway를 실패시킴

Ingress → Gateway API 실제 컷오버(release+prod) 중 실제로 겪음 — [[../decisions/2026-08-20-ingress-to-gateway-api-migration]] 참고.

## 증상

`Gateway`가 계속 `Accepted: False, reason: Invalid, message: "Check Gateway Events for more information."`로 실패. ALB Controller 로그를 보면:

```
"error":"TargetGroup port is empty. When using Instance targets, your service must be of type 'NodePort' or 'LoadBalancer'"
```

## 원인

`TargetGroupConfiguration.spec.defaultConfiguration.targetType`을 명시적으로 지정 안 하면 `instance`로 자동 추론된다("If unspecified, it will be automatically inferred as instance" — CRD 스키마 설명 그대로). `instance` 타입은 ALB가 **NodePort**로 노드에 접속해서 그 노드가 트래픽을 파드로 전달하는 방식이라, 대상 Service가 반드시 `NodePort` 또는 `LoadBalancer` 타입이어야 한다 — `ClusterIP` 서비스는 애초에 NodePort가 없어서 이 방식 자체가 불가능하다.

Grafana Alloy Faro 수집 엔드포인트(`alloy-faro` Service, `monitoring` 네임스페이스)가 정확히 이 경우였다 — `ClusterIP` 타입인데 `TargetGroupConfiguration`에 `targetType`을 안 넣어서(오늘 Ingress의 `faro_ingress`도 `target-type` annotation이 없었어서 똑같이 재현될 잠재 버그였음 — backend/frontend Ingress는 명시적으로 `target-type: ip`를 줬었기 때문에 안 걸렸음) Gateway 전체가 `Accepted: False`로 실패했다.

**중요**: 이건 Gateway API 자체의 새 버그가 아니라, **원래 있던 잠재 버그**였다 — `alloy-faro`/`faro_ingress`/`module.alloy_faro`는 오늘 이전엔 한 번도 실제로 끝까지 배포된 적이 없어서(다른 pre-existing drift로 남아있던 항목) Ingress로도 이 문제를 겪을 기회 자체가 없었을 뿐이다.

## 해결

`TargetGroupConfiguration`에 `targetType: ip`를 명시 — Pod IP로 직접 라우팅하는 방식이라 Service 타입과 무관하게 항상 동작함(오늘 backend/frontend TargetGroupConfiguration이 이미 이렇게 하고 있었음).

```yaml
apiVersion: gateway.k8s.aws/v1
kind: TargetGroupConfiguration
spec:
  targetReference:
    kind: Service
    name: alloy-faro
  defaultConfiguration:
    targetType: ip   # 이 줄이 없으면 ClusterIP 서비스에서 실패
```

## 덧붙임 — targetType 고치고 나서 또 걸린 헬스체크 경로 문제

`targetType: ip`로 고쳐서 `Gateway`는 `Accepted: True`가 됐는데, 실제 타겟그룹 헬스체크는 계속 `unhealthy`였음. `kubectl port-forward`로 alloy-faro 파드에 직접 붙여서 확인:

```
GET /            -> 404  (기본 헬스체크 경로 — Grafana Alloy의 faro.receiver는 이 경로를 아예 안 받음)
GET /collect     -> 405  (실제 트래픽 경로지만 GET 자체를 허용 안 함 — POST 전용, 헬스체크로 못 씀)
GET /-/ready     -> 200  (Grafana Alloy 자체의 표준 준비 상태 엔드포인트 — 이 포트에서도 정상 응답)
```

`TargetGroupConfiguration.spec.defaultConfiguration.healthCheckConfig.healthCheckPath: /-/ready`로 지정해서 해결. 이 계열의 컴포넌트(Grafana Alloy/Prometheus 등)는 `/-/ready`, `/-/healthy` 같은 표준 엔드포인트를 갖고 있는 경우가 많으니, 실제 트래픽 경로가 GET을 안 받는 컴포넌트를 헬스체크 대상으로 붙일 땐 이런 전용 엔드포인트가 있는지부터 확인할 것.

## 재발 방지

- **`TargetGroupConfiguration`을 새로 만들 때는 `targetType`을 항상 명시적으로 `ip`로 지정할 것** (이 클러스터의 서비스는 전부 ClusterIP이고 IRSA/CNI 구조상 `ip` 모드가 항상 맞음 — `instance` 모드를 쓸 이유가 없음). 기본값에 기대지 말 것.
- `Gateway`가 `Accepted: False`로 멈춰있으면 `kubectl describe gateway`보다 **ALB Controller 로그**(`kubectl logs -n kube-system deploy/aws-load-balancer-controller`)에서 실제 reconcile 에러를 먼저 확인할 것 — Gateway 오브젝트 자체의 이벤트/메시지는 "Check Gateway Events for more information"처럼 뭉뚱그려져 있어서 원인 파악에 도움이 안 됨.

## 관련
- [[../decisions/2026-08-20-ingress-to-gateway-api-migration]]
