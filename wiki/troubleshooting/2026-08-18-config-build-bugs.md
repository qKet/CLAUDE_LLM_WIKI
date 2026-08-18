---
title: 2026-08-18 설정/빌드 버그 3건 — SES output 오참조, notification-config 유령 ConfigMap, backend Dockerfile UID 충돌
category: troubleshooting
tags: [terraform, ses, configmap, docker, ci]
created: 2026-08-18
updated: 2026-08-18
---

# 2026-08-18 설정/빌드 버그 3건

서로 관련은 없지만 같은 날 각각 겪은 작은 설정/빌드 버그 3건. 2026-08-13 패턴과 동일하게 한 문서에 묶어서 기록.

## 1. `03_registry` output이 존재하지 않는 `module.ses` 참조

### 증상
`03_registry`에서 `terraform plan`이 곧바로 실패:
```
Error: Reference to undeclared module
  on outputs.tf line 38, in output "ses_identity_arn":
  38:   value       = module.ses.identity_arn
```

### 원인
SES는 모듈이 아니라 [ses.tf](../../../Infra/03_registry/ses.tf)에 raw 리소스(`aws_ses_domain_identity.this`)로 직접 선언돼 있는데, output이 존재하지 않는 `module.ses`를 참조하고 있었음. 처음엔 모듈로 만들 계획이었다가 raw 리소스로 바뀌면서 이 output만 안 고쳐진 것으로 보임 — 실제로 이 output이 어디서도 `remote_state`로 참조되지 않는 죽은(dead) output이라 지금까지 아무도 이 plan을 안 돌려봐서 안 드러났던 것.

### 해결
```hcl
output "ses_identity_arn" {
  value = aws_ses_domain_identity.this.arn   # module.ses.identity_arn → 이걸로 교체
}
```

### 재발 방지
Terraform은 참조 대상이 실제로 apply될 때까지 존재 여부를 검증 안 함 — output처럼 아무도 안 쓰는 코드는 `terraform plan`을 실제로 돌려보기 전까진 잘못된 참조가 몇 달이고 안 드러날 수 있다는 걸 보여준 사례. 방치된 output/변수는 주기적으로 plan 통과 여부라도 확인할 것.

## 2. `notification-config` — 만드는 코드가 없는 유령 ConfigMap 참조

### 증상
backend 파드가 `CreateContainerConfigError`로 못 뜸:
```
Warning  Failed  kubelet  Error: configmap "notification-config" not found
```

### 원인
`CD/helm/values.yaml`의 `backend.configMaps`에 `notification-config`가 추가돼 있었는데, 이 이름의 ConfigMap을 만드는 Terraform 리소스가 **어디에도 없었음**. 확인해보니 알림 기능에 필요한 값(`NOTIFICATION_QUEUE_URL`, `OPEN_ALERT_QUEUE_URL`)은 이미 [`04_data/main.tf`](../../../Infra/04_data/main.tf)의 `kubernetes_config_map.app_config`에 들어있었고, `application.yml`이 기대하는 env var 이름과도 정확히 일치했음 — 즉 새 ConfigMap 자체가 필요 없었는데, 새 알림 기능을 추가하면서 "새 ConfigMap이 필요하겠지"라고 참조만 추가하고 실제로 만들진 않은 것으로 보임.

### 해결
`backend.configMaps`에서 `notification-config` 제거(`CD/helm/values.yaml`, `values-prod.yaml`). 알림 값은 이미 `app-config`로 공급되고 있어서 그걸로 충분함.

### 재발 방지
새 ConfigMap/Secret을 Helm `values.yaml`에 참조로 추가할 때, 그걸 실제로 만드는 Terraform 리소스가 있는지 먼저 확인할 것 — 참조만 추가하고 실체를 안 만들면 Helm/ArgoCD는 아무 경고 없이 sync까지는 성공하고, 실제 파드가 뜨는 시점에야 `CreateContainerConfigError`로 드러남.

## 3. backend Dockerfile — Ubuntu Noble 베이스 전환으로 `useradd --uid 1000` 실패

### 증상
CI 빌드가 이 스텝에서 실패:
```
RUN useradd --system --uid 1000 --shell /usr/sbin/nologin appuser
→ useradd: UID 1000 is not unique
→ exit code: 4
```

### 원인
`backend/Dockerfile`이 `FROM eclipse-temurin:21-jre`(OS 버전 미고정)를 쓰고 있었는데, 이 태그의 베이스가 최근 **Ubuntu Noble(24.04)** 로 바뀜. Ubuntu 24.04 공식 이미지부터는 기본 `ubuntu` 유저가 이미 UID 1000을 점유하고 있어서(2024년에 많은 Dockerfile을 깨뜨린 걸로 알려진 변경), 뒤이은 `useradd --uid 1000`이 충돌함. 이 non-root 유저 생성 자체는 같은 날 팀원이 컨테이너 보안 강화 목적으로 추가한 것([[2026-08-18-config-build-bugs]] 작성 시점 기준 최신 커밋 `925110c`) — 의도는 맞았는데 베이스 이미지 태그가 하필 같은 시기에 바뀐 게 겹침.

### 해결
```dockerfile
FROM eclipse-temurin:21-jre-jammy   # Ubuntu 22.04 베이스로 고정 — ubuntu 기본 유저가 없어서 UID 1000 그대로 씀
```
실제로 `docker run eclipse-temurin:21-jre-jammy`로 UID 1000이 비어있는 것과 `useradd` 성공을 직접 검증함.

### 재발 방지
프로덕션 Dockerfile의 `FROM`은 OS 버전까지 명시적으로 고정하는 게 안전함 — `21-jre`처럼 마이너 태그만 쓰면 벤더가 베이스 OS를 바꿔도 아무 알림 없이 다음 빌드부터 깨질 수 있음. `frontend/Dockerfile`(`node:20-alpine`)은 이미 배포판까지 고정돼 있어서 이 문제가 없음 — backend도 같은 습관으로 맞춤.

## 관련
- [[loadtest-10000-open-run-cascading-failures]]
- [[../decisions/2026-08-18-capacity-planning-large-traffic-readiness]]
