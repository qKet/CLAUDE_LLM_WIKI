---
title: ALB Controller가 group.name을 바꿔도 옛 IngressGroup(ALB/TG)을 자동으로 안 치움
category: troubleshooting
tags: [alb, ingress, aws-load-balancer-controller]
created: 2026-08-11
updated: 2026-08-11
---

# ALB Controller가 `group.name`을 바꿔도 옛 IngressGroup(ALB/TG)을 자동으로 안 치움

## 증상

Grafana/ArgoCD Ingress를 각자 다른 IngressGroup(`qket-grafana`/`qket-argocd`)으로 처음 만들었다가, 나중에 하나(`qket-admin`)로 합치기로 결정하고 `group.name`을 바꿔서 apply했다([[../architecture/admin-ingress-shared-alb]]). 새 ALB(`team5-qket-admin-alb`)는 잘 만들어졌는데, **옛 ALB 2개(`team5-qket-grafana-alb`, `team5-qket-argocd-alb`)와 그 Target Group들이 그대로 AWS에 남아있었다**.

## 원인

ALB Controller 로그를 확인해보니 새 그룹(`qket-admin`)에 대한 "create model" 이벤트만 있고, 옛 그룹(`qket-grafana`/`qket-argocd`)을 정리하는 "delete" 이벤트는 전혀 없었다. ALB Controller의 reconcile 루프는 **현재 Ingress 오브젝트들이 선언하는 IngressGroup만 보고 그 상태를 만든다** — Ingress의 `group.name`이 바뀌면 컨트롤러 입장에선 "이 Ingress가 새 그룹에 속하게 됐다"만 인식할 뿐, "예전에 속했던 그룹은 이제 아무도 참조 안 하니 지워야 한다"는 역방향 정리는 하지 않는다. Kubernetes 오브젝트 자체가 지워질 때(Ingress 삭제)는 finalizer로 정리가 되지만, **오브젝트는 그대로 있고 소속 그룹만 바뀌는 경우**는 이 정리 경로를 안 탄다.

## 해결

`kubectl get ingress -A -o json`으로 클러스터 전체에서 옛 `group.name`(`qket-grafana`, `qket-argocd`)을 참조하는 Ingress가 하나도 없는 걸 먼저 확인한 뒤, AWS CLI로 옛 ALB 2개와 Target Group 2개를 수동 삭제.

## 재발 방지

- **기존 Ingress의 `group.name`을 바꾸는 작업은 "생성"이 아니라 "이관"으로 취급한다** — apply 후 반드시 AWS 콘솔이나 CLI로 옛 그룹 이름의 ALB/TG가 남아있는지 확인하는 걸 체크리스트에 넣는다.
- ALB Controller 로그(`kubectl logs -n kube-system deploy/aws-load-balancer-controller`)에서 "create"만 있고 "delete"가 없다면 옛 리소스가 고아로 남을 가능성을 의심한다.
- 이런 종류의 이관이 잦다면, `group.name`을 바꾸기 전에 먼저 Ingress를 완전히 지웠다가(옛 그룹이 정상적으로 finalizer 경로로 정리되는 걸 확인) 새 `group.name`으로 다시 만드는 2단계 방식이 더 안전할 수 있다 — 지금은 그렇게까진 안 하고 수동 확인으로 대응 중.

## 관련
- [[../architecture/admin-ingress-shared-alb]]
- [[destroy-order-incident-and-webhook-orphans]]
