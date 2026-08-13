---
title: KEDA 기반 backend 오토스케일링 구조
category: architecture
tags: [keda, hpa, autoscaling, metrics-server, kubernetes]
created: 2026-08-13
updated: 2026-08-13
---

# KEDA 기반 backend 오토스케일링 구조

## 구조 / 요약

`qket-backend`는 plain HPA가 아니라 **KEDA**(Kubernetes Event-Driven Autoscaling)로 오토스케일링됨.

- **엔진 설치**(Infra 레포, Terraform): `modules/addons/keda` → `02_k8s-addon`에서 `helm_release`로 KEDA 컨트롤러 설치
- **스케일 규칙**(CD 레포, Helm): `CD/helm/templates/backend-scaledobject.yaml`에 `ScaledObject` 정의 — `qket-backend` Deployment를 스케일 대상으로, 현재는 `cpu` 트리거(request 대비 70% 목표, min 4 / max 8)만 사용
- **필수 전제조건**: `metrics-server`(`modules/addons/metrics-server`)가 클러스터에 설치돼 있어야만 `cpu` 트리거가 동작함 — 없으면 `ScaledObject`도 내부 HPA도 정상 생성되지만 지표를 못 가져와서 영구히 스케일 안 함 ([[../troubleshooting/keda-scaling-missing-metrics-server]] 참고)

KEDA는 내부적으로 `ScaledObject`마다 표준 HPA를 자동 생성함(`keda-hpa-<scaledobject명>`) — `cpu` 트리거는 사실상 일반 HPA와 동일하게 동작하지만, KEDA를 쓰면 나중에 트리거만 추가해서(`redis`, `sqs` 등) 큐 길이 기반 스케일로 확장 가능. HPA로 시작했다가 나중에 KEDA로 갈아타는 것보다, 처음부터 KEDA 하나로 통일하는 게 낫다는 판단으로 채택.

`ScaledObject`를 Terraform(Infra)이 아니라 Helm(CD)에 둔 이유: `qket-backend` Deployment를 스케일하는 "애플리케이션 설정"이라 그 Deployment랑 같이 관리되는 게 맞다고 판단 — Terraform은 KEDA "엔진"을 클러스터에 설치하는 것까지만 담당.

## 스케일 판단 기준 — request 대비 평균 사용률, limit 아님

CPU 트리거는 **limit이 아니라 pod의 CPU request(500m) 대비 평균 사용률**을 5분 단위로 봄. 순간적으로 CPU limit(2 CPU)까지 튀는 버스트가 있어도, 그게 짧으면 5분 평균으로는 request 대비 낮게 잡혀서 스케일 트리거가 안 걸릴 수 있음 — 즉 **"순간 CPU 스로틀링이 심하다"와 "KEDA가 스케일아웃해야 한다"는 서로 다른 신호**임. 자세한 실측 사례는 [[../troubleshooting/backend-cpu-throttling-and-scaling-load-test]] 참고.

## ArgoCD와의 충돌 방지

`qket-cd-app.yaml`(ArgoCD Application, Terraform `kubectl_manifest`로 관리)에 `spec.ignoreDifferences`로 `Deployment/qket-backend`의 `/spec/replicas`를 명시적으로 무시하도록 설정해둠 — 이게 없으면 ArgoCD가 GitOps sync 때마다 KEDA가 늘린 replica 수를 Git의 값(고정값)으로 계속 되돌리려고 해서 KEDA와 ArgoCD가 서로 싸우게 됨.

## 의도적으로 어색하거나 특이한 지점

- `CD/helm/values.yaml`의 `backend.replicas`는 이제 **초기값일 뿐** — 실제 운영 중 replica 수는 KEDA가 관리하므로 이 값과 `kubectl get pods` 결과가 달라도 버그 아님
- `redis` 트리거를 나중에 추가할 걸 염두에 두고 만들었지만, 2026-08-13 기준 아직 `cpu` 트리거만 실제로 씀 — Redis 기반 대기열 길이 스케일링은 설계만 있고 미구현

## 관련
- [[../troubleshooting/keda-scaling-missing-metrics-server]]
- [[../troubleshooting/backend-cpu-throttling-and-scaling-load-test]]
- [[../troubleshooting/hikaricp-connection-storm-load-test]]
