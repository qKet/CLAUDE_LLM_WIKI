---
title: dev/Grafana/CD 접근을 VPN 필수로 바꿀지 — 논의만 하고 보류
category: decisions
status: 논의중
date: 2026-08-11
author: 이채영
tags: [security, vpn, wireguard, alb, access-control]
---

# dev/Grafana/CD 접근을 VPN 필수로 바꿀지 — 논의만 하고 보류

> ⚠️ 이 문서는 **결정이 아니라 논의 중단 시점의 스냅샷**이다. 사용자가 "팀원들이랑 더 상의를 해보고 결정해야할 사안"이라며 명시적으로 논의를 보류했다 — 다음에 이 주제를 다시 꺼낼 때 처음부터 다시 고민하지 않도록 지금까지 나온 옵션과 기울었던 방향만 기록해둔다. **팀 논의 없이 이 문서 내용을로 구현을 진행하면 안 된다.**

## 배경

[[../architecture/admin-ingress-shared-alb]]로 Grafana/ArgoCD를 IP 허용목록(`inbound-cidrs`)으로 제한했지만, "IP 허용목록"은 각 팀원의 IP가 고정이 아니면(집/카페/이동통신 등으로 바뀌면) 매번 Terraform 코드를 고쳐야 하는 방식이라 확장성에 한계가 있다는 문제의식에서 시작. `prod`는 그대로 공개로 두고, `dev`/Grafana/CD처럼 운영 도구성 접근만 VPN을 거치게 하고 싶다는 요구.

## 논의된 옵션

- **VPN 방식**: AWS Client VPN vs WireGuard(자체 호스팅)
  - AWS Client VPN — 관리형이라 운영 부담은 적지만, 시간당 연결 요금 + 접속 시간 요금 구조라 4인 소규모 팀 예산에 비해 비용이 크다고 판단해 **기울지 않음**
  - WireGuard — 이미 떠있는 bastion EC2에 설치하는 방향으로 **기울었음**(추가 인프라 비용 없음, 이미 있는 SSH bastion과 같은 역할 확장)
- **네트워크 구조**: 지금처럼 인터넷에 노출된 ALB에 IP 제한 vs **완전 비공개(internal ALB)** — VPN을 거쳐야만 해당 서브넷에 도달 가능한 internal ALB로 바꾸는 방향으로 **기울었음**(IP 허용목록보다 근본적으로 더 안전 — 애초에 인터넷에서 라우팅이 안 됨)

## 왜 보류됐나

사용자가 방향성(WireGuard + internal ALB) 자체엔 관심을 보였지만, 이게 팀 전체의 워크플로우(각자 로컬에 WireGuard 클라이언트 설정, 매번 VPN 연결 후 작업하는 습관 변화)에 영향을 주는 결정이라 **혼자 정하지 않고 팀원들과 상의 후 결정**하겠다고 명시적으로 중단했다.

## 다음에 이 논의를 재개할 때 참고할 것

- 위 두 축(VPN 방식, internal vs public+IP제한)에서 기울었던 방향은 WireGuard + internal ALB였지만, 이건 **논의였지 결정이 아니다** — 팀 상의 결과에 따라 완전히 다른 방향(예: 그냥 IP 허용목록 유지)이 나올 수 있다.
- 지금 상태([[../architecture/admin-ingress-shared-alb]])는 그 자체로 "그럭저럭 안전한 임시 상태"라 VPN 도입이 급하지 않다 — 팀 상의가 늦어져도 당장 보안 공백이 생기는 건 아니다.
- 만약 WireGuard + internal ALB로 실제 결정이 나면: bastion에 WireGuard 서버 설치, internal-facing ALB로 전환(`scheme: internal`), VPN 클라이언트 서브넷을 ALB가 있는 프라이빗 서브넷과 라우팅 연결하는 작업이 필요 — 아직 코드/설계 상세는 없음.

## 관련
- [[../architecture/admin-ingress-shared-alb]]
