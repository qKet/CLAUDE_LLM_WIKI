---
title: Ingress → Gateway API 마이그레이션 — 전체 완료(release/prod/admin), dev도 admin 전용으로 전환
category: decisions
status: 완료 — Ingress 오브젝트 전부 삭제(qket-release/qket-prod/argocd/monitoring 전체)
date: 2026-08-20
author: Claude Code
tags: [gateway-api, ingress, alb, aws-load-balancer-controller, terraform, helm, external-dns]
updated: 2026-08-20
---

> ✅ 2026-08-20 같은 날 후속: 아래 "1단계"로 시작해서 release 테스트 호스트네임 파일럿까지 검증한 뒤, **원래 계획(release 먼저, 검증 후 prod)과 달리 release+prod를 한 번에 완전히 컷오버**했다 — 사용자 판단: "Prod는 아직 올리지도 않아서(실서비스 오픈 전) 실 트래픽 리스크가 없으니 한 번에 가고 문제는 올리면서 해결하면 된다". `app_ingress_backend`/`app_ingress_frontend`/`faro_ingress` 3개 Ingress 리소스는 코드에서 완전히 삭제됨.
>
> ✅✅ 같은 날 한 번 더 후속: admin(Grafana/ArgoCD)도 이어서 마이그레이션 완료. 그리고 "개발 서버는 관리자만 들어가야 한다"는 결정에 따라 **dev.jun979.click(release)를 공개 ALB에서 admin Gateway로 재이동**시켜서 grafana/argocd/dev 셋이 팀원 IP 허용목록을 공유하게 함(prod는 공개 유지). `kubernetes_ingress_v1.grafana`/`argocd`도 완전히 삭제 — 이제 이 프로젝트에 Ingress 오브젝트가 하나도 없음. 최종 구현/검증 결과는 맨 아래 "admin(Grafana/ArgoCD) + dev 재이동 (같은 날 후속)" 섹션 참고.

# Ingress → Gateway API 마이그레이션

## 배경

`alb.ingress.kubernetes.io/*` annotation이 10개 넘게 쌓이면서 관리 부담이 커짐 — 특히 `healthcheck-path`가 Ingress 오브젝트 단위로만 적용돼서, backend/frontend Ingress를 어쩔 수 없이 둘로 쪼갠 것([[../architecture/admin-ingress-shared-alb]]에서도 같은 제약으로 Grafana/ArgoCD Ingress를 나눔)이 대표적인 pain point. 업스트림도 Ingress 대신 Gateway API를 권장하는 방향이라 전환하기로 함.

실도메인 4개(`dev.jun979.click`/`app.jun979.click`/`grafana.jun979.click`/`cd.jun979.click`)가 걸린 마이그레이션이라 처음엔 단계별로 진행하기로 했었음:

- **1단계(완료)**: CRD 설치 + `release` 환경에 테스트 호스트네임(`gw-dev.jun979.click`)으로 파일럿 — 실 트래픽 영향 0
- **2단계(완료, 당초 계획에서 범위 확장됨)**: `dev.jun979.click`(release) + `app.jun979.click`(prod) **동시** 실제 컷오버 — prod가 아직 실서비스 오픈 전이라 release만 먼저 하고 검증할 이유가 없어져서 범위를 합침
- 3단계(미착수): admin(Grafana/ArgoCD) — `inbound-cidrs`에 대응하는 필드(`LoadBalancerConfiguration.spec.sourceRanges`)가 실제로 존재함을 확인했으니(아래 "당초 예상과 달랐던 점" 참고) 갭은 아니지만, admin 도구라 보안 영향이 더 커서 가장 나중으로 미룸

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

## release+prod 완전 컷오버 (같은 날 후속)

파일럿 검증 직후, 사용자 판단으로 "release만 먼저"가 아니라 release+prod를 한 번에 진행. 실제로 겪은 것:

### 모듈을 4개로 재구성 (env별 for_each + 공유 리소스 분리)

`gateway-api-pilot` 모듈을 release/prod 양쪽에서 `for_each`로 인스턴스화하는 구조로 바꿈:

```
module.gateway_api_crds   (CRD + GatewayClass, 여전히 1개, 의존성 없음)
        ↓
module.alb_controller
        ↓
module.gateway_api_faro   (신설 — alloy-faro ReferenceGrant + TargetGroupConfiguration, 1개만)
        ↓
module.gateway_api_app    (구 gateway_api_pilot, for_each = local.ingress_config → release/prod 2개 인스턴스)
```

`gateway-api-faro`를 별도 모듈로 뺀 이유: Grafana Alloy Faro 수집 엔드포인트(`alloy-faro` Service)는 release/prod가 **같은 Service를 공유**하는데(모니터링은 환경 구분 없이 하나), 이걸 env별 `gateway-api-app` 인스턴스 안에 넣으면 `monitoring` 네임스페이스 안에 같은 이름의 `ReferenceGrant`/`TargetGroupConfiguration`을 release·prod 두 helm_release가 동시에 만들려다 소유권 충돌이 남(`referencegrants.gateway.networking.k8s.io "..." already exists"` 에러로 실제로 겪음). 공유 리소스는 한 번만, env별 리소스(Gateway/HTTPRoute/LoadBalancerConfiguration/backend·frontend TargetGroupConfiguration)는 네임스페이스 자체가 달라서 이름이 겹쳐도 안전.

Helm 릴리즈 이름도 env를 넣어 유일하게 만듦(`gateway-api-app-release`/`gateway-api-app-prod`) — 안 그러면 같은 `kube-system` 네임스페이스에 같은 릴리즈 이름 두 개가 충돌.

### 새로 겪은 버그 — TargetGroupConfiguration targetType 기본값이 ClusterIP Service에서 실패

`alloy-faro` Service(`ClusterIP`)용 `TargetGroupConfiguration`에 `targetType`을 명시 안 했다가(오늘 Ingress의 `faro_ingress`도 `target-type` annotation이 없었던 걸 그대로 반영한 것) `Gateway`가 `Accepted: False`로 실패함 — 자세한 내용은 [[../troubleshooting/gateway-api-instance-target-type-clusterip-service]]. `targetType: ip`로 고쳐서 해결. **이건 Gateway API의 새 버그가 아니라 원래 있던 잠재 버그** — `alloy-faro`/`faro_ingress`가 이날 이전엔 한 번도 실제로 끝까지 배포된 적이 없어서(다른 이유로 pending 상태였음) Ingress로도 이 문제를 겪을 기회 자체가 없었을 뿐.

### 다운타임

`dev.jun979.click`/`app.jun979.click` 둘 다 Ingress→Gateway API 전환 순간 몇 분(ALB provisioning + DNS 전파)의 다운타임이 있었음 — release는 실제 팀 트래픽이 있는 환경이라 사전에 사용자에게 고지하고 진행. "옛 Ingress destroy 먼저 → 새 Gateway apply 나중"으로 나누면 오히려 순차 실행돼서 다운타임이 더 길어지므로, 같은 apply 안에서 병렬로 처리(자세한 논의는 이 문서 초안 작성 대화 참고, 별도 기록은 안 함).

### 검증 결과

- `GatewayClass Accepted: True` (공유), release/prod `Gateway` 둘 다 실제 ALB(`team5-qket-gw-release-alb`/`team5-qket-gw-prod-alb`) 생성
- `HTTPRoute`: release는 backend/frontend/faro 전부 `ResolvedRefs: True`. prod는 `qket-backend-service`/`qket-frontend-service`가 아직 배포 전이라(실서비스 오픈 전) `BackendNotFound` — Gateway/ALB 자체는 정상, 이후 팀이 prod에 앱을 배포하면 자동으로 resolve될 예정
- ExternalDNS가 `dev.jun979.click`/`app.jun979.click` A/AAAA 레코드를 새 ALB로 실제로 UPSERT함(Route53에서 직접 확인)
- 기존 `app_ingress_backend`/`app_ingress_frontend`/`faro_ingress` 3개 리소스는 코드에서 완전히 제거, 옛 ALB(`team5-qket-alb`)의 관련 규칙도 정상적으로 사라짐

### 트레이드오프 / 남은 리스크

- `ssl-redirect`(HTTP→HTTPS)는 `RequestRedirect` HTTPRoute filter로 구현, 실제 302/301 리다이렉트 동작까지는 아직 브라우저로 직접 확인 안 함(TargetGroup/DNS 레벨 검증까지만 완료) — 다음에 실제로 브라우저 접속해서 확인 필요
- prod의 backend/frontend가 실제 배포된 뒤 `HTTPRoute ResolvedRefs`가 자동으로 `True`로 바뀌는지 재확인 필요
- 3단계(admin — Grafana/ArgoCD) 전부 미착수

## admin(Grafana/ArgoCD) + dev 재이동 (같은 날 후속)

release+prod 컷오버 직후, 사용자 지시로 admin도 이어서 진행. 추가로 "개발 서버는 관리자만 들어가야 한다"는 판단에 따라 dev(release)를 아예 공개 ALB에서 admin Gateway로 옮겨서 세 도메인(grafana/argocd/dev)이 IP 허용목록을 공유하게 함(prod는 실서비스용이라 공개 유지).

### 5번째 모듈: `gateway-api-admin`

```
module.gateway_api_crds → module.alb_controller → module.gateway_api_faro → module.gateway_api_admin
                                                                            → module.gateway_api_app (for_each: prod만)
```

- Gateway 하나(`qket-gw-admin`, `monitoring` 네임스페이스)에 **리스너 3개**(grafana-https/argocd-https/dev-https, 전부 포트 443) + http:80 리다이렉트 전용 리스너 — 오늘 Ingress의 `group.name` 공유와 동일한 효과를 리스너 여러 개로 표현
- `LoadBalancerConfiguration.spec.sourceRanges`로 팀원 IP 허용목록 적용(오늘 `inbound-cidrs`와 동일 — [[../architecture/admin-ingress-shared-alb]] 참고, 이 결정 자체는 그 문서의 트레이드오프를 그대로 계승)
- 인증서 3개(grafana/argocd/dev)를 **SNI로 한 443 리스너에 다중 연결** — `LoadBalancerConfiguration.spec.listenerConfigurations[].defaultCertificate`(1개) + `.certificates`(나머지 목록)로 가능. ALB가 SNI로 Host 헤더 보고 알아서 맞는 인증서 골라줌
- `HTTPRoute`는 각자 자기 백엔드와 같은 네임스페이스에 둠(grafana→monitoring, argocd→argocd, dev→qket-release) — Gateway 자신은 monitoring에 있어서 `allowedRoutes.namespaces.from: All`로 cross-namespace HTTPRoute 첨부를 허용해야 했음(이건 backendRef cross-namespace용 ReferenceGrant와는 별개 메커니즘 — Route가 다른 네임스페이스의 Gateway에 붙는 것 자체는 Gateway의 `allowedRoutes`가, Route의 backendRef가 다른 네임스페이스를 가리키는 것은 ReferenceGrant가 각각 담당)
- dev의 `/collect`(Faro) cross-namespace backendRef는 기존 `module.gateway_api_faro`의 ReferenceGrant를 그대로 재사용(release 네임스페이스를 이미 허용해뒀음 — 새로 안 만들어도 됨)

### 실제 컷오버 중 겪은 것 — Helm uninstall 타임아웃 + Terraform state 불일치

release를 공개 `gateway-api-app`에서 admin으로 옮기면서, 예전 release 전용 helm_release(`gateway-api-app-release`)를 삭제해야 했는데 `Error: uninstallation completed with 1 error(s): context deadline exceeded`가 남 — ALB/타겟그룹이 실제로 정리되는 데 Helm의 기본 대기시간보다 오래 걸린 것으로 보임. 실제로는 그 뒤 클러스터를 직접 확인해보니 Gateway/ALB 자체는 정상적으로 삭제 완료돼있었고, `TargetGroupConfiguration` 오브젝트 2개만 고아로 남아있었음(`kubectl delete`로 수동 정리) — Terraform state에는 이 helm_release가 여전히 "존재"로 남아있어서 `terraform state rm`으로 직접 정리해야 했음(재시도하면 이미 지워진 걸 또 지우려다 비슷하게 걸릴 걸로 예상돼서 재시도 대신 이 방법 택함).

### 검증 결과

`https://dev.jun979.click/`(200)/`/api/health`(200)/HTTP→HTTPS 리다이렉트(301), `https://grafana.jun979.click/`(302, 로그인 리다이렉트라 정상), `https://cd.jun979.click/`(200) 전부 팀원 허용 IP에서 확인. 타겟그룹 5개(argocd/grafana/backend/frontend/faro) 전부 healthy. AWS 보안그룹을 직접 조회해서 4개 팀원 IP가 80/443 양쪽에 정확히 반영된 것도 확인함.

**이 시점에서 `kubectl get ingress -A`가 빈 목록** — 이 프로젝트에 Ingress 오브젝트가 하나도 안 남음.

## 관련
- [[../troubleshooting/alb-controller-gatewayapi-boot-time-crd-check]]
- [[../architecture/admin-ingress-shared-alb]]
- [[../troubleshooting/crd-not-yet-installed-on-fresh-apply]]
- [[../troubleshooting/eks-destroy-layer-separation]]
- [[../runbook/daily-infrastructure-toggle]]
