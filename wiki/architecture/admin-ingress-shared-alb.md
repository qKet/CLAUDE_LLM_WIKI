---
title: Grafana + ArgoCD — IP 허용목록 공유 ALB
category: architecture
tags: [alb, ingress, grafana, argocd, monitoring, security]
created: 2026-08-11
updated: 2026-08-20
---

> ⚠️ 2026-08-20: 이 문서는 **Ingress 기반 구조(과거)** 를 설명한 것 — Gateway API로 완전히 이관되면서 `kubernetes_ingress_v1.grafana`/`argocd`는 삭제됐다. 지금은 `dev.jun979.click`(release)도 같이 합류해서 셋이 하나의 admin Gateway/ALB(`team5-qket-gw-admin-alb`)와 IP 허용목록을 공유한다 — 자세한 내용은 [[../decisions/2026-08-20-ingress-to-gateway-api-migration]] 참고. 아래 내용 중 "IngressGroup"/`group.name`/`inbound-cidrs` 같은 Ingress 전용 개념은 Gateway API에서 각각 "하나의 Gateway에 여러 리스너"/`LoadBalancerConfiguration.spec.sourceRanges`로 대체됐지만, **"IP 허용목록이 그룹 전체에 공유 적용된다"는 핵심 트레이드오프 자체는 새 구조에서도 그대로 유효**하다.

# Grafana + ArgoCD — IP 허용목록 공유 ALB

## 구조 / 요약

Grafana(`grafana.jun979.click`)와 ArgoCD UI(`cd.jun979.click`)는 팀 내부 관리 도구라, 공개 앱(`app_ingress_backend`/`app_ingress_frontend`)과는 별도로 **IP 허용목록으로 접근을 제한한 ALB 하나**를 공유한다. 코드는 `02_k8s-addon/admin-ingress.tf`.

- 각각 `kubernetes_ingress_v1.grafana`/`argocd`로 별도 Ingress 오브젝트(healthcheck-path가 서로 다르기 때문 — 아래 참고)
- `alb.ingress.kubernetes.io/group.name = "qket-admin"`를 둘 다 동일하게 줘서 **같은 IngressGroup**으로 묶임 → ALB Controller가 물리 ALB를 하나만 만듦(`load-balancer-name = "team5-qket-admin-alb"`), `group.order`(grafana=10, argocd=20)로 리스너 룰 우선순위만 나눔
- 각자 전용 ACM 인증서(`grafana.jun979.click`/`cd.jun979.click`) — 기존 dev/app 인증서가 와일드카드가 아니라서 재사용 불가, DNS 검증은 `aws_route53_record` + `aws_acm_certificate_validation`으로 완전 자동화
- `alb.ingress.kubernetes.io/inbound-cidrs = join(",", local.admin_allowed_cidrs)` — 팀원 IP만 나열한 리스트(현재: 팀원 1명 + 본인, `/32` 각각). 새 팀원이 추가되면 이 리스트에 한 줄만 추가하면 됨
- `healthcheck-path`는 각자 다르게: Grafana는 `/api/health`, ArgoCD는 `/healthz` — 둘 다 로그인 여부와 무관하게 항상 200을 주는 엔드포인트를 일부러 골랐음(아래 "의도적으로 특이한 지점" 참고)

## 의도적으로 어색하거나 특이한 지점

### `inbound-cidrs`는 Ingress 단위가 아니라 IngressGroup(공유 보안그룹) 단위로 적용된다

ALB Controller는 하나의 IngressGroup에 속한 여러 Ingress를 **하나의 ALB + 하나의 공유 보안그룹**으로 구현한다. `inbound-cidrs` 같은 annotation은 이 공유 보안그룹에 적용되므로, **같은 `group.name`을 쓰는 Ingress들은 전부 같은 IP 허용목록을 강제로 공유한다** — Grafana에만 다른 IP를 허용하고 ArgoCD는 더 좁게 주는 식의 세분화는 지금 구조로는 불가능하다. 실제로 처음엔 도구별로 다른 IP 정책을 주려고 Grafana/ArgoCD를 각자 다른 ALB로 분리했다가, "4인 팀 규모에선 '팀원이면 둘 다 접속 가능, 아니면 둘 다 차단'이면 충분하고 ALB 하나를 아낄 수 있다"는 판단으로 하나로 합쳤다(각자 다른 IP 정책이 필요해지면 다시 `group.name`을 나눠야 함 — 코드 주석에 이 트레이드오프를 남겨둠).

### healthcheck-path를 일부러 로그인-무관 엔드포인트로 골랐다

Grafana/ArgoCD 둘 다 기본 경로(`/`)는 로그인이 안 된 상태에서 302 리다이렉트를 반환한다. ALB 헬스체크는 200이 아니면 그 타겟을 unhealthy로 판단하므로, 기본 경로를 그대로 쓰면 **애플리케이션은 정상인데 ALB가 계속 unhealthy로 오판**하는 상황이 생긴다(백엔드 Ingress에서도 같은 계열의 문제를 겪었던 패턴). 그래서 로그인 여부와 무관하게 항상 200을 주는 전용 엔드포인트(`/api/health`, `/healthz`)를 골랐다.

### `healthcheck-path`가 Ingress 오브젝트 단위로만 적용된다는 제약이 애초에 Ingress를 둘로 나눈 이유

하나의 Ingress에 Grafana/ArgoCD 백엔드를 둘 다 넣으면 healthcheck 경로를 서로 다르게 줄 방법이 없다 — 이게 [[../decisions/2026-08-11-frontend-api-routing-alb-not-rewrites]]에서 backend/frontend Ingress를 나눈 이유와 완전히 같은 제약이다.

### ALB Controller는 IngressGroup을 바꿔도(`group.name` 변경) 옛 그룹을 자동으로 안 치운다

`group.name`을 바꾸면 논리적으로는 "이 Ingress가 이제 다른 그룹에 속한다"는 뜻이지만, ALB Controller는 옛 그룹에 남은 리소스(옛 ALB, Target Group)를 스스로 정리해주지 않는다 — 자세한 내용/실제 겪은 사고는 [[alb-ingressgroup-orphan-on-rename]].

## 관련
- [[../decisions/2026-08-11-frontend-api-routing-alb-not-rewrites]]
- [[../decisions/2026-08-11-monitoring-stack-design]]
- [[alb-ingressgroup-orphan-on-rename]]
- [[../decisions/2026-08-11-vpn-access-control-paused]]
