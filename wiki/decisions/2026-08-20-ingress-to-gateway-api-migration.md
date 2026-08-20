---
title: Ingress → Gateway API 마이그레이션 — 1단계(CRD/파일럿) 완료, 실 컷오버는 별도 단계
category: decisions
status: 1단계 구현 완료 (release 파일럿, 실 트래픽 영향 없음) — 2·3단계(실제 컷오버, admin) 미착수
date: 2026-08-20
author: Claude Code
tags: [gateway-api, ingress, alb, aws-load-balancer-controller, terraform, helm, external-dns]
updated: 2026-08-20
---

# Ingress → Gateway API 마이그레이션

## 배경

`alb.ingress.kubernetes.io/*` annotation이 10개 넘게 쌓이면서 관리 부담이 커짐 — 특히 `healthcheck-path`가 Ingress 오브젝트 단위로만 적용돼서, backend/frontend Ingress를 어쩔 수 없이 둘로 쪼갠 것([[../architecture/admin-ingress-shared-alb]]에서도 같은 제약으로 Grafana/ArgoCD Ingress를 나눔)이 대표적인 pain point. 업스트림도 Ingress 대신 Gateway API를 권장하는 방향이라 전환하기로 함.

실도메인 4개(`dev.jun979.click`/`app.jun979.click`/`grafana.jun979.click`/`cd.jun979.click`)가 걸린 마이그레이션이라 단계별로 진행하기로 확정:

- **1단계(이 문서, 완료)**: CRD 설치 + `release` 환경에 테스트 호스트네임(`gw-dev.jun979.click`)으로 파일럿 — 실 트래픽 영향 0
- 2단계(추후): 파일럿 검증 후 `dev.jun979.click` 실제 컷오버 → `prod`
- 3단계(추후): admin(Grafana/ArgoCD) — `inbound-cidrs`에 대응하는 필드(`LoadBalancerConfiguration.spec.sourceRanges`)가 실제로 존재함을 확인했으니(아래 "당초 예상과 달랐던 점" 참고) 갭은 아니지만, 그래도 가장 나중으로 미룸(admin 도구라 보안 영향이 더 큼)

## 결정

### 핵심 설계: CRD와 실제 오브젝트를 "같은 Helm 차트" 안에 넣고 `helm_release`로 설치

`kubernetes_manifest`/`kubectl_manifest`(GatewayClass/Gateway/HTTPRoute를 테라폼이 직접 만드는 방식)는 plan 단계에서 CRD가 클러스터에 이미 있는지 API 서버에 확인하는데, `02_k8s-addon`이 매일 밤 통째로 destroy→재생성되는 이 프로젝트 구조상 이게 매번 실패한다(ServiceMonitor/SecretStore로 이미 3일 연속 겪음 — [[../troubleshooting/crd-not-yet-installed-on-fresh-apply]]). `depends_on`을 걸어도 apply *순서*만 보장할 뿐 plan이 스키마를 조회하는 시점 자체는 못 늦춰줘서 소용없다.

`helm_release`는 테라폼이 "이 차트를 이 값으로 설치해라"는 선언만 diff할 뿐 차트 내용물의 kind/스키마를 plan 단계에서 확인하지 않는다 — 실제 적용은 apply 시 Helm이 처리하고, Helm은 자기 차트 안 `crds/` 폴더를 `templates/`보다 항상 먼저 설치하는 걸 스스로 보장한다. 그래서 CRD와 그걸 쓰는 오브젝트를 같은 차트에 넣으면 이 문제 자체가 원천적으로 발생하지 않는다.

### 모듈을 왜 3개로 쪼갰나 (당초 계획은 CRD+오브젝트 하나였다가, 실제로 겪고 나서 분리함)

처음엔 CRD+GatewayClass+Gateway+HTTPRoute+TargetGroupConfiguration을 전부 `gateway-api-crds` 모듈 하나에 넣고 `module.alb_controller` 다음에 apply하도록 했었다. 그런데 실제로 apply해보니 **AWS Load Balancer Controller가 자기 파드 부팅 시점에 딱 한 번 Gateway API CRD 존재 여부를 확인해서 `ALBGatewayAPI` 기능을 켤지 정한다**는 걸 발견함 — CRD가 컨트롤러보다 나중에 생기면, 컨트롤러는 "CRD 없음"으로 판단하고 그 파드가 살아있는 내내 Gateway API를 처리 안 한다(`kubectl rollout restart`로 재부팅해야만 정상화됨). 자세한 증상/원인은 [[../troubleshooting/alb-controller-gatewayapi-boot-time-crd-check]].

이 문제 때문에 모듈을 3개로 쪼갬:

```
module.gateway_api_crds   (CRD core 10종 + GatewayClass 싱글턴, 의존성 없음 — 제일 먼저 적용됨)
        ↓ depends_on
module.alb_controller     (이 시점에 부팅하면 CRD가 이미 있어서 ALBGatewayAPI가 정상적으로 켜짐)
        ↓ depends_on
module.gateway_api_pilot  (Gateway/HTTPRoute/LoadBalancerConfiguration/TargetGroupConfiguration —
                            alb_controller 자신의 차트가 제공하는 LoadBalancerConfiguration/
                            TargetGroupConfiguration CRD가 필요해서 그 다음이어야 함)
```

`gateway-api-crds`(CRD가 컨트롤러보다 먼저 필요)와 `gateway-api-pilot`(컨트롤러 자신의 CRD가 필요해서 컨트롤러보다 나중이어야 함)은 순서 요구사항이 정반대라 하나로 합칠 수 없었다.

### CRD 버전 관리: 의도적으로 IAM 정책과 다른 정책

ALB Controller의 IAM 정책(`data.http`로 `main` 브랜치 JSON을 매번 fetch)과 달리, Gateway API core CRD는 **특정 버전에 고정(vendoring)** — `kubernetes-sigs/gateway-api` v1.6.1 standard channel을 레포에 직접 커밋(`Infra/modules/addons/gateway-api-crds/chart/crds/gateway-api-standard-v1.6.1.yaml`). 스키마가 하루아침에 바뀌는 게 IAM 권한 하나 느는 것보다 훨씬 위험한 실패 모드라서 의도적으로 다른 정책을 씀.

AWS 전용 CRD(`LoadBalancerConfiguration`/`TargetGroupConfiguration`/`ListenerRuleConfiguration`)는 **여기서 따로 vendoring하지 않는다** — `module.alb_controller`가 설치하는 공식 `aws-load-balancer-controller` Helm 차트(v3.5.0)가 자기 `crds/` 폴더에 이미 포함해서 설치해준다는 걸 확인함(2026-08-20 — 처음엔 우리도 이 3개를 따로 vendoring했다가, 두 CRD의 설치 시각이 그 helm_release 시각과 정확히 일치하는 걸 보고 중복임을 알아채서 제거함). 중복 vendoring하면 두 차트가 서로 다른 버전을 밀어넣는 flip-flop 위험만 생기므로 제거가 맞는 방향.

### CD(ArgoCD)로 빼는 대안 — 기각

GatewayClass/Gateway/HTTPRoute를 Terraform(`kubernetes_manifest`) 대신 CD(ArgoCD가 관리하는 Helm 템플릿)로 빼는 방법도 검토했다. ArgoCD의 sync는 Terraform의 plan-time 스키마 검증을 안 거치므로 "생성 시" CRD 순서 문제는 확실히 풀린다. 하지만 실제 AWS ALB를 만드는 오브젝트가 Terraform state 밖에 있으면, 매일 밤 `module.alb_controller`가 destroy될 때 순서 보장이 깨져서 ALB가 고아로 남을 위험이 있다([[../troubleshooting/eks-destroy-layer-separation]] 참고 — 애초에 Ingress를 CD에서 Terraform으로 옮긴 이유와 동일한 문제). `helm_release`로 Terraform에 그대로 두면 여전히 state에 잡히므로 `depends_on`으로 destroy 순서도 지킬 수 있어 이 대안을 기각함.

## 1단계 실제 구현 (2026-08-20, 검증 완료)

- `Infra/modules/addons/gateway-api-crds/` — core CRD 10종 + `GatewayClass "alb"` (controllerName: `gateway.k8s.aws/alb`)
- `Infra/modules/addons/gateway-api-pilot/` — release 환경에 `Gateway`(호스트네임 `gw-dev.jun979.click`, HTTP:80만, TLS 없음 — 테스트용이라 전용 ACM 인증서를 안 만들었음) + `HTTPRoute`(`/api`→`qket-backend-service`, `/`→`qket-frontend-service`, 오늘 Ingress 2개가 하던 라우팅을 규칙 2개로 통합) + `LoadBalancerConfiguration`(scheme/tags/loadBalancerName) + `TargetGroupConfiguration` 2개(backend: 8081/`/actuator/health`, frontend: `/healthz`)
- `Infra/modules/addons/external-dns`에 `sources` 값 추가(`service`,`ingress`,`gateway-httproute`) — 차트 기본값(service/ingress만)이라 HTTPRoute를 아예 감시 안 했었음. `module.external_dns`에 `depends_on = [module.alb_controller, module.gateway_api_crds]` 추가 — CRD 없이 이 source가 켜지면 external-dns 파드가 크래시루프 날 위험이 있어서.
- 검증 결과: `GatewayClass Accepted: True`, `Gateway`가 실제 ALB(`team5-qket-gw-pilot-alb`) 생성 후 `Programmed: True`, `HTTPRoute ResolvedRefs/Accepted: True`, ExternalDNS가 Route53에 `gw-dev.jun979.click` A/AAAA alias 레코드 실제로 생성(`aws route53 list-resource-record-sets`로 확인) — 파이프라인 전체(CRD→GatewayClass→Gateway→ALB→HTTPRoute→TargetGroupConfiguration→ExternalDNS→Route53) 정상 동작 확인.
- 기존 `app_ingress_backend`/`app_ingress_frontend`(release/prod 전부, 실트래픽 담당)는 전혀 건드리지 않음.

## 당초 예상과 달랐던 점

- **`inbound-cidrs`(admin IP 허용목록) 갭이 없었음**: `LoadBalancerConfiguration` CRD 스키마를 직접 읽어보니 `spec.sourceRanges`("optional list of CIDRs that are allowed to access the LB")가 정확히 이 역할을 함. 처음 설계 검토 때는 "갭이라 별도 Security Group이 필요하다"고 판단했었는데, 실제 CRD를 fetch해서 확인해보니 아니었음 — 3단계(admin 마이그레이션) 때 그대로 쓰면 됨.
- **ALB Controller의 부팅 시점 CRD 체크**(위 "모듈을 왜 3개로 쪼갰나" 참고)는 사전 조사로는 전혀 예상 못 했던 부분 — 실제로 apply해보고 나서야 발견함.

## 트레이드오프 / 남은 리스크

- 파일럿은 TLS 없이 HTTP:80만 검증함 — 2단계(실제 컷오버) 때는 `gw-dev.jun979.click` 이 아니라 실제 도메인용 인증서를 `LoadBalancerConfiguration.spec.listenerConfigurations[].defaultCertificate`(ACM ARN)로 연결해야 함
- `ssl-redirect`(HTTP→HTTPS 리다이렉트)의 Gateway API 쪽 정확한 메커니즘은 아직 미확인 — 2단계 착수 전에 `RequestRedirect` HTTPRoute filter 방식으로 스파이크 필요
- 2단계(실제 `dev.jun979.click` 컷오버)/3단계(admin) 전부 미착수 — 이 문서는 1단계 범위만 다룸

## 관련
- [[../troubleshooting/alb-controller-gatewayapi-boot-time-crd-check]]
- [[../troubleshooting/crd-not-yet-installed-on-fresh-apply]]
- [[../troubleshooting/eks-destroy-layer-separation]]
- [[../architecture/admin-ingress-shared-alb]]
- [[../runbook/daily-infrastructure-toggle]]
