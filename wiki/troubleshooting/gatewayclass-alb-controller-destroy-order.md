---
title: GatewayClass를 CRD 차트에서 분리 — ALB Controller와 생성/파괴 순서가 정반대로 필요했던 문제
category: troubleshooting
tags: [gateway-api, alb-controller, terraform, destroy, finalizer]
created: 2026-08-22
updated: 2026-08-22
---

# GatewayClass를 CRD 차트에서 분리 — ALB Controller와 생성/파괴 순서가 정반대로 필요했던 문제

## 증상

`02_k8s-addon` destroy 중 `module.gateway_api_crds.helm_release.gateway_api_crds`가
`Still destroying...`를 계속 반복 — [[ebs-csi-addon-destroyed-before-dev-datastore-pvc]]와
증상은 똑같은데, 이번엔 EBS CSI/KEDA가 아니라 **ALB Controller**가 원인이었음.

## 원인

`modules/addons/gateway-api/crds`는 원래 Gateway API core CRD와 `GatewayClass` 싱글턴을 **같은
Helm 차트**에 묶어서 설치하고 있었음 — 이유는 "CRD와 그걸 쓰는 GatewayClass가 module.alb_controller
(ALB Controller)보다 반드시 먼저 존재해야 한다"는 요구사항(ALB Controller는 부팅 시점에 딱 한 번
Gateway API CRD 존재 여부를 확인해서 자기 기능을 켤지 정함 — 2026-08-20에 실제로 겪은 문제) 때문에
`module.alb_controller` 쪽에서 `depends_on = [module.gateway_api_crds]`를 걸어뒀음.

문제는 `GatewayClass`가 **ALB Controller 자신의 finalizer**(`gateway.k8s.aws/gatewayclass`)를
갖고 있다는 것 — 이 finalizer는 ALB Controller가 살아있어야만 처리(제거)될 수 있음. 그런데
`depends_on`은 destroy 시 항상 create의 역순이라, `module.alb_controller`가
`module.gateway_api_crds`보다 destroy될 때는 **먼저** 사라짐(create 땐 나중에 만들어졌으니까) —
그 결과 GatewayClass를 지우려 할 시점엔 이미 그 finalizer를 처리해줄 컨트롤러가 없어서
`Still destroying...`이 영원히 반복됨.

즉 "생성은 CRD/GatewayClass가 컨트롤러보다 먼저"와 "파괴는 GatewayClass가 컨트롤러보다 먼저"라는
**서로 반대 방향의 순서 요구사항이 한 리소스(GatewayClass)에 동시에 걸려있었던 것** — 같은 차트
안에 있는 한 `depends_on` 하나로는 둘 다 만족시킬 수 없음.

## 해결

`GatewayClass`를 `modules/addons/gateway-api/crds` 차트에서 완전히 빼서, `02_k8s-addon/main.tf`에
독립된 `kubectl_manifest.gateway_class` 리소스로 분리하고 `depends_on = [module.alb_controller]`
(CRD 쪽과 반대 방향)을 걺:

- **생성**: `gateway_api_crds`(CRD) → `alb_controller` → `gateway_class` — 문제없음(GatewayClass는
  컨트롤러가 이미 있어야 실제로 쓰이니 오히려 순서상 자연스러움)
- **파괴**: `gateway_class`(컨트롤러 살아있을 때 finalizer 정상 처리) → `alb_controller` →
  `gateway_api_crds`(이제 GatewayClass가 없어서 finalizer 걱정 없이 깨끗하게 삭제됨)

`Gateway` 오브젝트를 실제로 만드는 `module.gateway_api_app`/`gateway_api_admin`에도
`kubectl_manifest.gateway_class`를 `depends_on`에 추가 — 이 둘은 GatewayClass를 참조하는
쪽이라 그 생성 완료를 기다려야 함(기존엔 `module.gateway_api_crds`만 의존하고 있어서, 분리 후엔
GatewayClass 생성 완료가 보장 안 됐었음).

부수적으로 발견한 별개 문제: `modules/addons/gateway-api/crds/chart/templates/`에 잘못 들어있던
untracked 파일 `gateway.yaml`(`pilot` 모듈의 파일이 실수로 복사되어 들어간 것 — 이전 세션에서
이미 "잘못 놓인 파일 같다"고 확인만 하고 삭제는 안 했었음)이 `helm template` 렌더링 자체를
깨뜨리는 걸 확인(`.Values.pilot.enabled`가 이 차트엔 정의 안 돼있어서 nil pointer 에러) — 삭제해서
해결. 이 파일이 언제부터 실제 apply에 영향을 줬는지는 불명확하나, 방치했으면 다음 fresh apply에서
`module.gateway_api_crds` 자체가 실패했을 것.

## 재발 방지

- [[ebs-csi-addon-destroyed-before-dev-datastore-pvc]]의 "재발 방지" 체크리스트에 있는 일반 원칙에
  더해: **한 컨트롤러가 관리하는 CRD와, 그 컨트롤러 자신이 finalizer를 붙이는 싱글턴 오브젝트(이번
  경우 GatewayClass)는 절대 같은 Helm 차트/모듈에 묶지 말 것** — 생성 순서(CRD가 컨트롤러보다 먼저)와
  파괴 순서(싱글턴이 컨트롤러보다 먼저)가 구조적으로 반대일 수 있음. 이런 싱글턴은 항상 별도
  리소스로 분리해서 컨트롤러 쪽에 `depends_on`을 걸 것.
- 로컬 Helm 차트를 수정할 때는 `helm template <chart-dir>`로 렌더링이 실제로 되는지 확인하는
  습관을 들일 것 — untracked/잘못 복사된 파일이 조용히 다음 apply를 깨뜨릴 수 있음(git status로
  untracked 파일이 있는지도 주기적으로 확인).

## 관련
- [[ebs-csi-addon-destroyed-before-dev-datastore-pvc]]
- [[alb-controller-gatewayapi-boot-time-crd-check]]
- [[../decisions/2026-08-20-ingress-to-gateway-api-migration]]
