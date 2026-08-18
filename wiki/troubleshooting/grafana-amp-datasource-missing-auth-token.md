---
title: Grafana의 grafana-amazonprometheus-datasource 플러그인이 쿼리에 인증 헤더를 안 붙임 (해결됨)
category: troubleshooting
tags: [monitoring, grafana, amp, prometheus, sigv4, 해결됨]
created: 2026-08-12
updated: 2026-08-18
---

# Grafana의 grafana-amazonprometheus-datasource 플러그인이 쿼리에 인증 헤더를 안 붙임 (해결됨)

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

## 추가 조사 (2026-08-12, 2차 — 더 파봄)

첫 조사 이후 사용자가 "플러그인 버그 다시 파보자"고 해서 재조사함. systematic-debugging 절차대로 새 증거를 모으고 가설을 세워 하나씩 검증함.

7. **URL 끝 슬래시(trailing slash) 가설 — 기각**: `amp_query_endpoint`(AWS가 주는 `prometheus_endpoint`)가 `.../ws-xxxx/`처럼 끝에 `/`가 붙어서 나오는데, 플러그인이 여기 `api/v1/query`를 이어붙이면 `//`(이중 슬래시)가 생겨서 API Gateway가 라우트를 못 찾는 것 아닌가 하는 가설을 세움. boto3로 이중 슬래시 URL에 정상 서명해서 요청해보니 **똑같이 403 Missing Authentication Token**이 나와서 처음엔 가설이 맞는 것처럼 보였음. 근데 실제로 Terraform에서 `trimsuffix()`로 슬래시를 제거하고 재적용(Pod 완전 재시작까지 포함)했더니 **queryData는 여전히 그대로 실패했고, 오히려 이전엔 통과하던 checkHealth까지 같이 깨짐**. 즉 이중 슬래시는 원인이 아니었고, "Missing Authentication Token"이라는 메시지 자체가 AWS API Gateway가 라우트 불일치든 인증 헤더 누락이든 가리지 않고 똑같이 반환하는 범용 에러라 우연히 같은 문구가 나온 것으로 판명. **원래 URL(끝 슬래시 있는 채)로 원복함.**
8. **결정적 증거 — 인증 헤더를 아예 안 붙이고 요청**: boto3로 SigV4 서명을 전혀 안 하고 순수 `requests.get()`만 날려봤더니 **정확히 동일한 `403 Missing Authentication Token`**이 나옴. 이게 가장 직접적인 증거 — AWS가 "서명이 틀렸다"가 아니라 "서명 자체가 없다"고 말하는 상황이라는 뜻. 즉 플러그인의 `queryData` 요청에는 애초에 `Authorization` 헤더가 안 붙고 있을 가능성이 매우 높음.
9. **authType `ec2_iam_role` 옵션도 시도 — 기각**: 플러그인 문서를 보니 인증 방식으로 `default`/`keys`/`credentials`/`ec2_iam_role` 4가지가 있고, 지금까지 `default`(IRSA로 될 줄 알았음)와 `keys`(access key)만 시도했었음. `ec2_iam_role`은 IRSA/EC2 인스턴스 프로파일용으로 별도 존재하길래 Terraform 안 거치고 Grafana API로 임시 데이터소스를 만들어 빠르게 테스트함 — **역시 동일하게 실패**(checkHealth 403, queryData Missing Auth Token). 이걸로 인증 방식 3가지(default/keys/ec2_iam_role) 전부 동일 실패 확인.
10. **업스트림에 이미 보고된 알려진 버그 발견**: GitHub `grafana/grafana-amazonprometheus-datasource` 저장소의 [Issue #640](https://github.com/grafana/grafana-amazonprometheus-datasource/issues/640) — 제목이 정확히 "Missing Authentication Token until 'Save and Test' is Selected". **Helm으로 프로비저닝한 데이터소스에서 우리와 똑같은 증상**을 겪고 있고(UI에서 수동으로 "Save and Test"를 누르면 그때만 해결되는데, 우리처럼 `additionalDataSources`로 provisioning하면 UI 편집 자체가 막혀있어서 이 workaround를 쓸 수가 없음), 이슈는 **아직 미해결(Waiting) 상태**로 남아있음. 즉 우리 환경 특이 문제가 아니라 플러그인 자체의 알려진 미해결 버그.

## 해결 (2026-08-18)

**원인**: SigV4 서명을 실제로 요청에 붙이려면 두 군데를 **같이** 켜야 하는데, 그중 하나(데이터소스 쪽 `jsonData.sigV4Auth`)가 빠져있었음.

- 이미 켜뒀던 것: Grafana 서버 레벨 `GF_AUTH_SIGV4_AUTH_ENABLED`(위 3번 시도) — 근데 이것만으로는 부족했음
- **빠져있던 것**: 데이터소스 `jsonData`에 `sigV4Auth: true`를 명시적으로 켜는 것. `authType: "default"`만 설정했었는데, 이 전용 플러그인은 `authType`과 별개로 `sigV4Auth` 필드가 있어야 실제로 서명 미들웨어를 태우는 것으로 보임(플러그인 자체 문서/코드에 명시적으로 안 나와있어서 우리도 처음엔 놓쳤음).

GitHub Issue #640의 코멘트(`erezzarum`, 2026-05)에서 정확히 같은 조합을 워크어라운드로 제시했고, 다른 사용자(`shivam-dubey-1`)도 효과를 확인해줬던 것을 뒤늦게 발견 — 이슈 자체는 여전히 Open이라 공식 수정은 아니지만, 이 설정 조합으로 우회 가능함이 확인됨.

**실제 적용한 값** (`modules/addons/monitoring/main.tf`):
```hcl
set {
  name  = "grafana.grafana\\.ini.auth.sigv4_auth_enabled"
  value = "true"
}

# 기존 authType/defaultRegion에 추가
set {
  name  = "grafana.additionalDataSources[0].jsonData.sigV4Auth"
  value = "true"
}

set {
  name  = "grafana.additionalDataSources[0].jsonData.sigV4Region"
  value = var.aws_region
}
```

적용(`terraform apply -target=module.monitoring`) 후 Grafana pod 재시작, `/api/ds/query`로 AMP 데이터소스 직접 쿼리 테스트 → `status: 200`, 5일 전 데이터까지 정상 조회됨. **미해결로 뒀던 6일 만에 해결.**

## 재발 방지

이 AMP 전용 플러그인은 인증 관련 필드가 여러 개(`authType`, `sigV4Auth`, `sigV4Region`) 있고 문서화가 약해서, "이 정도면 됐겠지" 하고 하나만 켜고 넘어가기 쉽다. 비슷한 AWS SigV4 연동 플러그인을 또 붙일 일이 있으면, **서버 레벨 설정과 데이터소스 레벨 설정을 항상 세트로 확인**할 것 — 서버만 켜고 데이터소스 필드를 빠뜨리면 이번처럼 `checkHealth`는 통과하는데 실제 쿼리만 계속 실패하는, 원인 특정이 어려운 상태가 됨.

## 현재 상태

- Grafana에서 AMP 데이터소스로 직접 조회 가능 — 클러스터 재생성 이전의 과거 데이터도 대시보드에서 바로 볼 수 있음
- `remoteWrite`(저장)와 `queryData`(조회) 둘 다 정상 동작 확인됨

## 관련
- [[../decisions/2026-08-11-monitoring-stack-design]]
- [GitHub Issue #640](https://github.com/grafana/grafana-amazonprometheus-datasource/issues/640) — 동일 증상, 워크어라운드 코멘트 출처(이슈 자체는 아직 Open)
