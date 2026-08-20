---
title: AWS Load Balancer Controller가 부팅 시점에 딱 한 번 Gateway API CRD를 확인해서 기능을 켤지 정함
category: troubleshooting
tags: [alb, gateway-api, aws-load-balancer-controller, terraform, helm]
created: 2026-08-20
updated: 2026-08-20
---

# ALB Controller가 부팅 시점에 딱 한 번 Gateway API CRD를 확인해서 기능을 켤지 정함

[[../decisions/2026-08-20-ingress-to-gateway-api-migration]] 1단계(release 파일럿) 작업 중 실제로 겪음.

## 증상

Gateway API CRD(`gateway-api-crds` 모듈)와 GatewayClass/Gateway/HTTPRoute 파일럿 오브젝트를 모두 정상적으로 apply했는데도, 몇 분이 지나도록 `GatewayClass`가 `ACCEPTED: Unknown`, `Gateway`가 `PROGRAMMED: Unknown`에서 전혀 진행이 안 됨. ALB Controller 로그를 보면:

```
{"level":"info","msg":"Disabling ALBGatewayAPI: missing required CRDs","missing":["Gateway","GatewayClass","HTTPRoute","GRPCRoute"]}
```

CRD는 이미 클러스터에 있는데(`kubectl get crd`로 확인됨) 컨트롤러는 계속 "없다"고 말하는 상태.

## 원인

AWS Load Balancer Controller는 **자기 파드가 부팅하는 시점에 딱 한 번** Gateway API CRD(`gateway.networking.k8s.io` 그룹)가 클러스터에 등록돼 있는지 확인해서, 있으면 `ALBGatewayAPI` 기능을 켜고 없으면 꺼버린다. 이 판단은 그 파드가 살아있는 동안 다시 안 하는 **일회성 체크**다.

우리 원래 설계(2026-08-20 오전 1차 버전)는 `module.gateway_api_crds`가 `module.alb_controller` **다음에** apply되도록 `depends_on`을 걸어놨었다 — "GatewayClass/Gateway 등을 실제로 처리할 컨트롤러가 먼저 있어야 한다"는 생각이었는데, 이게 정반대였다. 순서가 이렇게 되면:

1. `module.alb_controller` apply → 컨트롤러 파드가 뜨면서 부팅 시점 CRD 체크 → 아직 CRD가 없으니 `ALBGatewayAPI` 비활성화 결정, 그대로 굳어버림
2. `module.gateway_api_crds` apply → CRD는 이제 생겼지만, 이미 떠서 돌고 있는 컨트롤러 파드는 그 사실을 모름(재확인 안 함)
3. GatewayClass/Gateway를 아무리 만들어도 그 컨트롤러가 처리를 안 하니 영원히 `Unknown`

`kubectl rollout restart deployment/aws-load-balancer-controller -n kube-system`으로 파드를 재부팅하면, 그 시점엔 CRD가 이미 있으니 `ALBGatewayAPI`가 켜지고("starting gateway route reconciler" 로그 확인) 즉시 정상 동작(GatewayClass Accepted, Gateway가 실제 ALB 주소를 받아옴)한다.

## 해결

`depends_on` 방향을 반대로 뒤집었다 — CRD가 컨트롤러보다 **먼저** apply되도록:

```
module.gateway_api_crds  (의존성 없음, 제일 먼저 적용 가능)
        ↓ depends_on
module.alb_controller    (이 시점에 부팅하면 CRD가 이미 있어서 ALBGatewayAPI가 정상적으로 켜짐)
        ↓ depends_on
module.gateway_api_pilot (GatewayClass/Gateway/HTTPRoute 등 실제 오브젝트 — 컨트롤러의
                           LoadBalancerConfiguration/TargetGroupConfiguration CRD도 필요해서
                           alb_controller 다음이어야 함)
```

이렇게 3단으로 모듈을 쪼갠 이유가 여기 있다 — `gateway-api-crds`(CRD+GatewayClass, alb_controller보다 먼저)와 `gateway-api-pilot`(실제 오브젝트, alb_controller보다 나중)은 요구하는 순서가 정반대라 하나로 합칠 수 없었다. 자세한 모듈 구조는 [[../decisions/2026-08-20-ingress-to-gateway-api-migration]] 참고.

이 순서를 지켜두면 **매일 아침 재적용될 때마다** `module.gateway_api_crds`가 `module.alb_controller`보다 먼저 apply되므로, 컨트롤러 파드가 뜨는 시점엔 항상 CRD가 이미 있다 — `kubectl rollout restart`를 매일 수동으로 할 필요가 없다.

## 재발 방지

- **ALB Controller가 CRD 유무로 기능을 켜고 끄는 다른 경우가 또 있을 수 있다** — 이번에 겪은 건 Gateway API였지만, 로그에 `Disabling NLBGatewayAPI`/`Disabling GatewayListenerSet`도 같이 찍혀있었다(TLSRoute/TCPRoute/UDPRoute/ListenerSet CRD 관련, 우리가 안 쓰는 기능이라 미확인). 이 컨트롤러가 의존하는 CRD를 추가할 땐 항상 "컨트롤러 부팅보다 먼저 있어야 하나?"를 먼저 확인할 것.
- 이런 종류의 "부팅 시점 1회 감지" 버그는 로그에 반드시 단서가 남는다 — `kubectl logs -n kube-system deploy/aws-load-balancer-controller`에서 `Disabling`으로 시작하는 줄이 있으면 이 계열을 의심할 것.
- Terraform의 `depends_on`은 여기서도 apply *순서*만 보장한다 — 순서 자체를 반대로 설계하지 않으면 아무리 `depends_on`을 걸어도 소용없다는 점에서 [[crd-not-yet-installed-on-fresh-apply]]와 근본적으로 같은 교훈("plan/boot 시점에 필요한 것은 apply 순서 설계로 미리 맞춰둬야 한다")이다.

## 관련
- [[../decisions/2026-08-20-ingress-to-gateway-api-migration]]
- [[crd-not-yet-installed-on-fresh-apply]] — CRD 관련 순서 문제의 다른 계열(Terraform plan-time 스키마 검증)
