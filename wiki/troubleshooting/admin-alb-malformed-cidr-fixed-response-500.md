---
title: admin_allowed_cidrs에 /32 빠진 IP 하나가 dev.jun979.click 전체를 500으로 내린 사고
category: troubleshooting
tags: [alb, gateway-api, security-group, cidr, admin-ingress]
created: 2026-08-21
updated: 2026-08-21
---

# `admin_allowed_cidrs`에 `/32` 빠진 IP 하나가 dev.jun979.click 전체를 500으로 내린 사고

## 증상

`https://dev.jun979.click/`, `/api/*` 전부 `net::ERR_HTTP_RESPONSE_CODE_FAILURE` / `HTTP 500`. 백엔드/프론트 파드는 `1/1 Running`, 재시작 0회로 완전히 멀쩡했고, `kubectl logs`에도 이렇다 할 에러가 없었음 — 애플리케이션 레벨 문제가 전혀 아니었음.

## 원인 진단

`aws elbv2 describe-rules`로 실제 ALB(`team5-qket-gw-admin-alb`)의 HTTPS 리스너 규칙을 직접 조회해서 발견:

```
dev.jun979.click, /api/*  →  fixed-response 500
dev.jun979.click, /*      →  fixed-response 500
```

`/api`(backend)와 `/`(frontend) 라우팅 규칙 자체가 **AWS Load Balancer Controller에 의해 자동으로 fixed-response 500 fallback으로 교체돼있었음** — HTTPRoute의 `ResolvedRefs`/`Accepted` 조건은 둘 다 `True`였지만(K8s 오브젝트 자체는 멀쩡해 보임), 실제 타겟그룹이 컨트롤러 레벨에서 못 만들어진 상태였음.

컨트롤러 로그(`kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller`)에서 결정적 증거 확인:
```
error: operation error EC2: AuthorizeSecurityGroupIngress, https response error StatusCode: 400,
api error InvalidParameterValue: CIDR block 58.29.99.44 is malformed
```

`Infra/02_k8s-addon/variables.tf`의 `admin_allowed_cidrs`(grafana/argocd/dev 공유 ALB 접속 허용 팀원 IP 목록)에 IP 하나(`58.29.99.44`, 준혁님)가 `/32` 없이 맨 IP로만 들어가 있었음. AWS가 이걸 malformed CIDR로 거부 → ALB Controller가 `qket-gw-admin` Gateway의 보안그룹 규칙 갱신(reconcile) 자체를 계속 실패 → 그 여파로 dev.jun979.click의 HTTPRoute들이 정상 타겟그룹에 못 붙고 fixed-response 500으로 대체됨.

`grafana.jun979.click`/`cd.jun979.click`은 영향 없었음 — 그쪽 rule은 이미 만들어진 타겟그룹을 참조하고 있어서 재조정 실패의 영향을 안 받았고, 하필 dev.jun979.click 라우팅이 그 시점에 새로 만들어지던 중이라 직격탄을 맞은 것으로 보임.

## 해결

```hcl
# variables.tf
"58.29.99.44"      → "58.29.99.44/32"
```
`terraform apply -target=module.gateway_api_admin` 즉시 적용 — ALB 룰이 forward로 바뀌고 타겟그룹 `healthy`, 사이트 200 확인.

## 재발 방지

- `admin_allowed_cidrs`에 팀원 IP를 추가할 때 **반드시 `/32` 포함**을 확인할 것 — 리스트 항목 형식 검증(예: `regex(...)` validation 블록)을 변수에 추가하는 것도 고려 가능.
- 이번처럼 "K8s 오브젝트(HTTPRoute)는 `Accepted: True`인데 실제로는 안 됨"이 재현되면, `kubectl describe`만으로는 못 잡고 **ALB Controller 파드 로그**와 **실제 ALB 리스너 규칙(`aws elbv2 describe-rules`)**을 직접 봐야 진짜 원인이 보인다는 걸 기억할 것 — Gateway API/ALB Controller 조합에서는 K8s 오브젝트 상태와 AWS 쪽 실제 반영 여부가 분리돼 있어서 이런 괴리가 생길 수 있음.

## 관련
- [[../architecture/admin-ingress-shared-alb]]
- [[../decisions/2026-08-20-ingress-to-gateway-api-migration]]
