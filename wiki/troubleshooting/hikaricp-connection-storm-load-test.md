---
title: 부하테스트만 돌리면 ArgoCD/Grafana까지 같이 멈춤 — HikariCP 풀 과다 설정으로 인한 "커넥션 생성 폭풍"
category: troubleshooting
tags: [monitoring, hikaricp, rds, connection-pool, t3-burstable, load-test, grafana, cloudwatch]
created: 2026-08-13
updated: 2026-08-13
---

# 부하테스트만 돌리면 ArgoCD/Grafana까지 같이 멈춤 — HikariCP 풀 과다 설정으로 인한 "커넥션 생성 폭풍"

## 증상

k6로 부하테스트(로그인 1000명)를 돌릴 때마다, 부하테스트 대상이 아닌 ArgoCD/Grafana 화면까지 같이 느려지거나 응답이 안 됨. 부하테스트를 멈추면 다시 정상으로 돌아옴.

## 원인 진단

처음엔 "RDS가 못 버티는 것"으로 의심했으나, CloudWatch로 RDS 지표를 직접 확인해보니 **RDS 자체는 전혀 힘들지 않았음**:
- CPU 사용률 9%대
- 실제 연결 수 51~61개 (여유 있음)

RDS가 원인이 아니라면 뭐가 문제였나 — HikariCP 설정을 다시 봤더니, 당시 `maximum-pool-size: 40`(레플리카당) × backend replica 8개 = **잠재적으로 최대 320개 연결**을 만들 수 있는 구성이었음. `db.t3.micro`의 `max_connections`는 공식 산정식(`{DBInstanceClassMemory/12582880}`)으로 약 85개뿐이라, 이론상 RDS 연결 한도 자체를 넘길 수 있는 위험한 설정이었음.

그런데 실측 연결 수(51~61개)는 한도를 안 넘었는데도 문제가 발생했다는 게 핵심 — **연결 개수가 아니라 "연결을 새로 만드는 동작" 자체가 원인**이었음:

- HikariCP 풀이 크면, 부하가 몰릴 때 여러 replica가 동시에 새 커넥션을 다수 생성하려 시도함 (TCP 핸드셰이크 + DB 인증)
- 이 커넥션 "생성" 자체가 CPU를 상당히 소모하는 동작
- 당시 워커 노드가 **t3.medium(버스터블 인스턴스)** 이었는데, 버스터블 인스턴스의 CPU 크레딧은 **노드 단위로 공유**됨 — 즉 backend 파드들이 커넥션 생성으로 CPU 크레딧을 몰아 쓰면, 같은 노드에 떠있던 ArgoCD/Grafana 파드까지 같이 CPU를 못 받아서 멈춘 것처럼 보인 것
- RDS CloudWatch 지표만 보면 전혀 안 드러나는 문제였음 — 노드 레벨 CPU 크레딧 소모라는, DB 지표와 무관한 층위의 병목이었기 때문

## 해결

`backend/src/main/resources/application.yml`의 HikariCP 설정을 축소:

```yaml
hikari:
  maximum-pool-size: ${DB_POOL_SIZE:20}
  minimum-idle: ${DB_POOL_SIZE:20}
  connection-timeout: 3000
```

`CD/helm/values.yaml`에 `backend.dbPoolSize: 10`으로 낮춰서 배포(`DB_POOL_SIZE` env var로 주입). `replica 수 × dbPoolSize`가 RDS `max_connections`(~85)를 넉넉히 안 넘도록 하는 게 기준.

## 재발 방지

- 부하테스트 전에 항상 `CD/helm/values.yaml`의 `backend.replicas`(또는 KEDA `maxReplicas`) × `backend.dbPoolSize`가 RDS 연결 상한을 넘지 않는지 계산해볼 것 — `Infra` 레포 `loadtest/README.md`(주의사항 섹션)에 이 체크리스트를 명시해둠
- "관련 없어 보이는 서비스까지 같이 느려진다"는 증상은 노드 단위 리소스(버스터블 CPU 크레딧, 메모리 등) 공유를 의심할 신호 — 대상 서비스의 지표만 보지 말고 노드 레벨 지표(`node_cpu_seconds_total` 등)도 같이 볼 것
- 이후 워커 노드를 t3.xlarge로 업그레이드하면서 버스터블 크레딧 고갈 리스크 자체는 완화됐지만, 근본 원인이었던 "커넥션 생성 폭풍" 자체는 풀 사이즈를 줄인 게 실질적 해결책

## 관련
- [[backend-cpu-throttling-and-scaling-load-test]]
- [[keda-scaling-missing-metrics-server]]
