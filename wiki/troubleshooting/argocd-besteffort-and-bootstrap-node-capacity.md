---
title: 부트스트랩 노드 하나가 ArgoCD를 포함한 인프라 파드를 혼자 떠받치던 문제 — 노드 1→2, ArgoCD resources.requests 부여
category: troubleshooting
tags: [argocd, qos, besteffort, karpenter, node-group, capacity]
created: 2026-08-21
updated: 2026-08-21
---

# 부트스트랩 노드 하나가 ArgoCD 포함 인프라 파드를 혼자 떠받치던 문제

## 배경

prod 2000명 e2e 부하테스트 중 [[backend-cold-start-cpu-contention-during-rollout]] 조사를 하다가, AWS 콘솔 "노드" 탭에서 고정 노드그룹 노드(`ip-10-70-2-63`, t3.large) 하나만 CPU 92%로 유독 높은 걸 발견 — Karpenter가 그때그때 만드는 노드들보다 오히려 이쪽이 더 빡빡했음.

## 원인 진단

`kubectl describe node`로 이 노드에 뜬 파드를 세어보니 **34개** — 대부분 ArgoCD 컴포넌트(`argocd-application-controller`, `argocd-applicationset-controller`, `argocd-dex-server` 등)였고, 전부 **CPU/메모리 requests가 0(`{}`)** — 즉 `BestEffort` QoS. 이건 2026-08-18 만 명 부하테스트 때 이미 문서화된 리스크([[loadtest-10000-open-run-cascading-failures]] "재발 방지" 2번)와 정확히 같은 구조 — 노드가 바빠지면 커널 스케줄러가 이 파드들부터 밀어내서 자기 헬스체크에도 제때 응답 못 하고 kubelet한테 강제 재시작당함.

동시에 이 노드(`01_infrastructure`의 `node_desired_size/min/max`)가 **1개로 고정**돼 있어서, 클러스터 제어 컴포넌트(ArgoCD 등)를 얹을 여유 노드 자체가 애초에 없었던 것도 근본 원인 중 하나로 판단.

## 해결

### 1. ArgoCD 컴포넌트에 최소 resources.requests 부여

`modules/addons/argocd/values/resources.yaml`(신규) — `controller`/`repoServer`/`server`/`applicationSet`/`dex`/`redis`/`notifications` 전부 requests+limits 지정(예: `repoServer: requests cpu:100m/memory:128Mi`). `modules/addons/argocd/main.tf`의 `helm_release.this`가 이 파일을 `values` 배열에 추가로 읽게 함.

`terraform apply -target=module.argocd` 후 `kubectl get pods -n argocd -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass`로 확인 — 상시 컴포넌트 전부 `BestEffort` → **`Burstable`**로 전환됨(`argocd-redis-secret-init`은 한 번 돌고 끝나는 Job이라 예외, 상시 리스크 대상 아님).

### 2. 부트스트랩 노드 1→2

`01_infrastructure/variables.tf`의 `node_desired_size`/`node_min_size`/`node_max_size`를 1→2로 상향(전부 고정값, 오토스케일링 범위 아님 — elastic scaling은 전부 Karpenter가 전담). `terraform apply -target=module.eks`로 반영, 새 노드 조인 확인.

## 재발 방지 / 남은 것

- ArgoCD 차트를 업그레이드할 때 `resources.yaml`의 컴포넌트 키가 여전히 유효한지(`helm show values argo/argo-cd --version X`) 확인 필요 — 차트 버전에 따라 경로가 바뀔 수 있음(이 모듈의 `main.tf` 상단 주석에 이미 같은 경고가 notifications 관련해서 있음).
- 노드 2개로 늘려도 여전히 고정값이라, 나중에 인프라 애드온이 더 늘어나면(신규 addon 등) 다시 빡빡해질 수 있음 — 그때 재검토 필요.

## 관련
- [[loadtest-10000-open-run-cascading-failures]]
- [[backend-cold-start-cpu-contention-during-rollout]]
- [[../architecture/cluster-autoscaler]]
