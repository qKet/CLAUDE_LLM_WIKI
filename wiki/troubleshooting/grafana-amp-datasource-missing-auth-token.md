---
title: Grafana의 grafana-amazonprometheus-datasource 플러그인이 쿼리에 인증 헤더를 안 붙임 (미해결)
category: troubleshooting
tags: [monitoring, grafana, amp, prometheus, sigv4, 미해결]
created: 2026-08-12
updated: 2026-08-12
---

# Grafana의 grafana-amazonprometheus-datasource 플러그인이 쿼리에 인증 헤더를 안 붙임 (미해결)

## 증상

Grafana에 AMP(Amazon Managed Prometheus)를 데이터소스로 추가했는데, 실제 PromQL 쿼리를 날리면 항상 이 에러가 남:

```
unexpected response with status code 403: {"message":"Missing Authentication Token"}
```

`checkHealth`(데이터소스 저장 시 연결 확인)는 `status=ok`로 통과하는데, 실제 패널/쿼리 요청(`endpoint=queryData`)만 이 에러가 남. instant 쿼리, range 쿼리 둘 다 동일하게 실패.

## 배경

목표: [[../decisions/2026-08-11-monitoring-stack-design]]에서 결정한 대로 지표 데이터를 AMP에 영구 저장(`remoteWrite`)하는 것까진 성공(`prometheus_remote_storage_samples_total`로 33만 건+ 전송, 실패 0건 확인됨). 그런데 Grafana가 그 데이터를 "다시 읽어오는" 쪽이 안 됨 — Prometheus는 AMP로 데이터를 계속 잘 보내고 있지만, Grafana 화면은 여전히 로컬(클러스터 안) Prometheus만 보고 있어서, EKS를 destroy/재생성하면 로컬 Prometheus가 새로 시작되면서 과거 그래프가 끊겨 보임. 이걸 풀려고 Grafana에 AMP를 직접 조회하는 데이터소스를 추가하려 함.

## 원인 진단 (여기까지 확인함)

아래 순서로 하나씩 좁혀나감:

1. **범용 `prometheus` 타입 + `sigV4Auth: true`로 시도** → `Credential should be scoped to correct service: 'aps'` 에러. Grafana의 범용 SigV4 서명 미들웨어가 AMP(`aps`) 서비스명을 인식 못 함.
2. **AWS 공식 전용 플러그인 `grafana-amazonprometheus-datasource`(v3.1.0)로 교체** → 에러가 `Missing Authentication Token`으로 바뀜(서명 자체를 시도조차 안 하는 것으로 보임).
3. **Grafana 서버 레벨 `GF_AUTH_SIGV4_AUTH_ENABLED=true` 추가** → 변화 없음(애초에 이 전용 플러그인엔 필요 없는 설정이었을 수 있음).
4. **직접 검증 — 제 로컬 AWS 자격증명(boto3)으로 같은 AMP 쿼리 API에 수동 SigV4 서명해서 요청** → **200 성공, 실제 데이터 정상 수신**. 즉 AMP 쿼리 API 자체, URL, IAM 권한(`aps:QueryMetrics` 등)은 전부 문제없음이 확인됨.
5. **Grafana Pod 안에서 IRSA 환경변수 확인** (`AWS_ROLE_ARN`, `AWS_WEB_IDENTITY_TOKEN_FILE`) → 정상 주입됨.
6. **진단용으로 IRSA 대신 access key 방식(`authType: keys`)의 임시 데이터소스를 새로 만들어서 같은 쿼리 시도** → **똑같이 `Missing Authentication Token`으로 실패**. IRSA든 access key든 자격증명 방식과 무관하게 동일하게 실패한다는 게 확인됨 — 이건 자격증명/권한 문제가 전혀 아니라는 뜻.

## 결론

**`grafana-amazonprometheus-datasource` v3.1.0 플러그인 자체의 버그(또는 이 버전에서의 알려진 제약)로 판단됨.** `checkHealth`는 통과하는데 실제 `queryData` 호출에서는 자격증명 종류와 무관하게 인증 헤더 자체를 안 붙이는 것으로 보임. IAM/IRSA/AMP API 쪽은 전부 정상 확인됐으므로, 우리 인프라 설정 문제가 아니라 플러그인 쪽 문제.

## 현재 상태 / 임시 조치

- **데이터 자체는 안전함** — `remoteWrite`는 정상 작동 중이라 Prometheus가 수집한 지표는 AMP에 계속 쌓이고 있고, EKS를 destroy/재생성해도 안 없어짐.
- **Grafana에서 AMP를 직접 조회하는 것만 아직 안 됨** — 지금 당장 과거 데이터를 봐야 하면 AWS 콘솔의 AMP 쿼리 화면에서 직접 PromQL로 조회 가능.
- 관련 Terraform 코드(`modules/monitoring`의 `additionalDataSources` 설정, `iam.tf`의 `grafana_amp_query` 정책)는 그대로 남겨둠 — 플러그인 버전을 바꾸거나 문제가 해결되면 바로 쓸 수 있는 상태.

## 다음에 시도해볼 것

- 플러그인 버전 다운그레이드/업그레이드 (`grafana.plugins`에 버전 명시: `grafana-amazonprometheus-datasource X.Y.Z`)
- GitHub의 `grafana-amazonprometheus-datasource` 이슈 트래커에서 같은 증상(`Missing Authentication Token` on queryData) 검색
- Grafana 자체 버전을 올려서 재시도 (지금 kube-prometheus-stack 기본 번들 버전 확인 필요)
- 최후 수단: AMP 앞단에 SigV4 서명을 대신 처리해주는 간단한 프록시(sidecar)를 두는 방법

## 관련
- [[../decisions/2026-08-11-monitoring-stack-design]]
