---
title: backend actuator 포트 분리(8081) 이후 Prometheus ServiceMonitor가 계속 예전 포트를 스크랩하던 문제
category: troubleshooting
tags: [monitoring, prometheus, servicemonitor, actuator, grafana]
created: 2026-08-19
updated: 2026-08-19
---

# backend actuator 포트 분리 이후 Prometheus ServiceMonitor가 계속 예전 포트를 스크랩하던 문제

## 증상

Grafana `qket` 대시보드에서 HikariCP 커넥션풀, HTTP 에러율, 엔드포인트별 요청수/응답시간 패널이 전부 비어있음. 부하테스트(2000명 규모) 도중 발견됨.

## 원인

[qKet/backend#29](https://github.com/qKet/backend/issues/29)로 actuator가 일반 요청과 스레드풀을 공유하던 문제를 고치면서, `management.server.port: 8081`로 actuator를 별도 포트로 분리함(이 이슈는 이미 팀원이 해결). 근데 이 변경이 반영된 배포([[loadtest-10000-open-run-cascading-failures]] 참고)와 별개로, **Prometheus `ServiceMonitor`는 여전히 예전 설정**(`targetPort: 8080`, `path: /api/actuator/prometheus`)을 그대로 참조하고 있었음 — actuator 포트 분리 작업 범위에 이 리소스가 안 들어가 있었던 것.

`kubectl port-forward`로 파드에 직접 붙여서 `:8081/actuator/prometheus`는 정상 응답(200)하는 걸 확인했고, Prometheus `/api/v1/targets`에서 이 job의 `health: down`, `lastError: "server returned HTTP status 404"`, `scrapeUrl: http://<pod-ip>:8080/api/actuator/prometheus`를 확인해서 원인을 특정함.

## 해결

`Infra/02_k8s-addon/main.tf`의 `kubernetes_manifest.backend_service_monitor`:

```diff
  endpoints = [
-   { targetPort = 8080, path = "/api/actuator/prometheus", interval = "15s" }
+   { targetPort = 8081, path = "/actuator/prometheus", interval = "15s" }
  ]
```

`targetPort=8081`로 바뀐 이유는 backend#29와 동일 — actuator가 서버 메인 포트(8080, context-path `/api`)와 별개로 관리 전용 포트(8081)에서 뜨고, **관리 포트는 `server.servlet.context-path`를 상속하지 않아서** `/api` 접두어도 같이 빠져야 함(`/actuator/prometheus`, `/api/actuator/prometheus` 아님).

`terraform apply -target=kubernetes_manifest.backend_service_monitor`로 좁게 적용, `up{job="qket-backend-service"}`가 다시 1로 돌아오는 것과 대시보드 데이터 복구를 확인함.

## 재발 방지

- **actuator 포트/경로를 바꾸는 변경은 backend 코드뿐 아니라 그걸 스크랩/헬스체크하는 쪽(ServiceMonitor, ALB Ingress healthcheck annotation, readiness/liveness probe)까지 전부 같이 확인할 것** — 이번에 ALB Ingress healthcheck(backend#29에서 이미 처리)와 readiness/liveness probe(Deployment에 이미 반영)는 맞았는데 ServiceMonitor만 빠져있었음. "actuator 포트를 참조하는 곳 목록"을 만들어두면 좋음: readinessProbe, livenessProbe, ALB Ingress `healthcheck-port`/`healthcheck-path`, Prometheus ServiceMonitor.
- Prometheus `up` 메트릭과 실제 대시보드 패널 데이터가 갑자기 비면, 대시보드 자체보다 **스크랩 타겟이 살아있는지**(`/api/v1/targets`, `lastError`)부터 확인하는 게 빠름.

## 관련
- [[loadtest-10000-open-run-cascading-failures]] — backend#29가 여기서 나온 발견 중 하나로 등록됨
- [[grafana-amp-datasource-missing-auth-token]] — 같은 계열(스크래핑/조회 설정 미스매치)의 이전 사례
