---
title: "/api 라우팅을 Next.js rewrites()에서 ALB path 라우팅으로 이전"
category: decisions
status: 확정
date: 2026-08-11
author: 이채영
tags: [alb, ingress, nextjs, frontend, backend]
---

# `/api` 라우팅을 Next.js `rewrites()`에서 ALB path 라우팅으로 이전

## 배경

[[../troubleshooting/cd-helm-chart-deploy-review]]의 문제 1을 고칠 때는 `frontend/next.config.js`의 `rewrites()`가 `/api/*` 요청을 `CLUSTER_IP`(K8s 내부 Service 주소)로 프록시하는 구조를 그대로 살리고, 빠져있던 env 값만 채워 넣었다. 이 방식이 배포 후 실제로 API 호출이 실패하는 프로덕션 버그를 냈다.

## 결정

`/api/*` 라우팅을 프론트 컨테이너의 책임에서 완전히 빼고, **ALB가 path 기반으로 backend/frontend를 직접 라우팅**하게 바꿨다.

- Ingress를 하나에서 둘로 분리: `kubernetes_ingress_v1.app_ingress_backend`(path `/api`, `group.order = 10`) / `app_ingress_frontend`(path `/`, `group.order = 20`) — `02_k8s-addon/main.tf`
- 두 Ingress가 healthcheck-path가 다르다는 것도 분리 이유 중 하나(backend는 헬스 엔드포인트, frontend는 `/`) — ALB의 `healthcheck-path` annotation은 Ingress 오브젝트 단위로 적용되기 때문에, 하나의 Ingress에 두 서비스를 같이 담으면 healthcheck 경로를 다르게 줄 방법이 없었음
- `group.name`을 같게 둬서(같은 IngressGroup) 두 Ingress가 물리적으로 **하나의 ALB**를 공유 — path만 다르게 라우팅되므로 도메인/ALB는 그대로 하나

## 왜 `rewrites()`가 문제였나

Next.js의 `rewrites()`는 **빌드 타임에 값을 정적으로 확정**한다. `CLUSTER_IP` 같은 값을 빌드 시점 환경변수로 주입해 프록시 destination을 만드는데, 이후 이미지가 재사용되거나 배포 파이프라인에서 빌드와 배포 시점이 어긋나면(이미지 캐시, 재배포 등) 실제 런타임에 필요한 값이 빌드 시점에 얼어붙은 값과 달라질 수 있다 — Kubernetes/컨테이너 이미지처럼 "한 번 빌드해서 여러 환경에 재사용"하는 배포 모델과 근본적으로 안 맞는 방식이었고, 실제로 이 불일치가 프로덕션에서 API 호출 실패로 나타났다.

ALB path 라우팅은 빌드 타임에 아무 값도 얼리지 않는다 — 라우팅 규칙은 Ingress(Terraform이 관리하는 K8s 오브젝트)에 있고, 이미지는 그냥 자기 자신의 요청만 처리하면 된다.

## 고려한 대안

- **`rewrites()` 유지 + 배포 파이프라인에서 항상 재빌드 강제** — 기각. 근본 원인(빌드 타임 고정)을 안 고치고 우회하는 방식이라 같은 클래스의 버그가 다른 형태로 재발할 여지가 남음.
- **`next.config.js`에서 런타임 값을 읽도록 변경(`env` 대신 API route handler로 프록시)** — 검토는 했으나, 결국 ALB가 이미 path 기반 라우팅을 네이티브로 지원하는데 애플리케이션 레이어에서 같은 일을 또 하는 건 중복 책임이라고 판단, ALB로 완전히 이관.

## 트레이드오프 / 남은 리스크

- Ingress가 backend/frontend 두 개로 늘어나 관리 포인트가 하나 더 생김(대신 `group.name` 공유로 ALB 자체는 여전히 하나).
- 로컬 개발 환경(`npm run dev`)에서는 ALB가 없으므로 여전히 `rewrites()` 또는 다른 프록시 방식이 필요할 수 있음 — 로컬/배포 환경의 라우팅 방식이 갈라진다는 점은 이후 온보딩 문서에 명시할 필요 있음(아직 안 됨).

## 관련
- [[../troubleshooting/cd-helm-chart-deploy-review]]
- [[../architecture/admin-ingress-shared-alb]]
