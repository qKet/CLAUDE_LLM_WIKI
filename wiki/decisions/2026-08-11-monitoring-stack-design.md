---
title: 모니터링 스택 설계 (Prometheus/Grafana/Loki)
category: decisions
status: 논의중
date: 2026-08-11
author: 이채영
tags: [monitoring, prometheus, grafana, loki, k8s-addon]
---

# 모니터링 스택 설계 (Prometheus/Grafana/Loki)

## 배경

지금까지는 지표/로그를 볼 방법이 없어서 `kubectl logs`/AWS 콘솔을 그때그때 뒤지는 식으로 트러블슈팅했다(바로 오늘 ALB 고아 리소스, ExternalDNS 레코드 문제를 전부 이 방식으로 찾았다). 이번에 모니터링/로그 스택을 제대로 구축하기로 함 — "지표", "DB 상태", "로그"가 서로 다른 도구가 필요한 별개 문제라는 걸 먼저 정리하고 시작.

## 결정 (지금까지 확정된 것)

| 항목 | 결정 |
|---|---|
| 지표 수집/시각화 | `kube-prometheus-stack` Helm 차트(Prometheus+Grafana+Alertmanager+node-exporter+kube-state-metrics) |
| 설치 위치 | `02_k8s-addon`(ArgoCD/ALB Controller와 같은 자리 — 클러스터 전역 addon, `04_data` 값 불필요) |
| 1차 모니터링 범위 | **클러스터 레벨만** — 노드 CPU/메모리, 파드 상태, ALB 트래픽. 앱 커스텀 지표(Spring Actuator/Micrometer)는 2차로 미룸 |
| DB(RDS/Redis) 상태 | 별도 exporter 없이 **Grafana에 CloudWatch를 데이터소스로 연결** — RDS/ElastiCache가 이미 CloudWatch에 지표를 보내고 있어서 추가 인프라 불필요 |
| 로그 저장 | **Loki**(+ Promtail/Alloy를 DaemonSet으로) — ELK/EFK 대신. 이유: 가벼움, Grafana 네이티브 통합, 4인 팀 규모에 ELK는 과함 |
| Loki 백엔드 스토리지 | **S3**(신규 전용 버킷, 기존 포스터 버킷과 분리) — EBS 대신 S3를 쓰면 AZ 고정/CSI 드라이버 고아 리스크 자체가 없음 |
| Loki 버킷/지표용 EBS 볼륨 위치 | **`03_registry`** — `02_k8s-addon`에 두면 매일 밤 destroy될 때 같이 날아감(실수로 여기 두자고 했다가 사용자가 바로 잡아냄). `03_registry`는 이미 "공유·불변·release/prod 구분 없음·절대 안 지움" 성격(ECR/OIDC)이라 로그/지표 영구 저장소도 같은 카테고리로 봐서 여기 배치. `04_data`는 workspace(release/prod)별로 두 번 생성되는 구조라 클러스터 전체가 공유하는 단일 저장소엔 안 맞음 |
| release/prod 로그를 한 버킷에 모아도 되는가 | 됨 — Promtail이 K8s namespace를 라벨로 자동 태깅(`namespace="qket-release"` 등)해서, 물리적으로 한 버킷에 있어도 Grafana에서 라벨로 완전히 분리해서 조회 가능(Prometheus가 이미 같은 방식으로 release/prod 지표를 한 저장소에서 다루는 것과 동일 원리) |
| Grafana 접속 방법 | **포트포워딩만** — ArgoCD UI 접속 방식과 동일 패턴(`kubectl port-forward`). 공개 도메인(`grafana.jun979.click`) 노출은 안 함 — 인증 강화가 안 된 상태에서 외부 노출은 위험 |
| 알림 채널 | Slack vs 이메일 — **미정, 팀이 실제로 뭘로 소통하는지에 따라 결정**(아래 미해결 참고) |

## 고려한 대안

- **로그: ELK/EFK vs Loki** — ELK가 풀텍스트 검색은 더 강력하지만 Elasticsearch 클러스터 운영 부담이 4인 팀 규모에 안 맞음. Loki 선택.
- **지표 영속성: EBS(동적 프로비저닝) vs S3(Loki 전용)** — 지표는 원래 EBS가 표준(Prometheus TSDB는 로컬 디스크 기반)이라 완전히 회피는 못 하지만, 로그(Loki)는 S3 백엔드가 표준 지원이라 EBS의 AZ 고정/CSI 고아 리스크를 아예 피할 수 있었음. 지표 쪽 EBS는 여전히 필요하면 AZ 고정 + node affinity로 대응하기로 함(아직 실제 구현은 안 함).
- **로그 버킷을 기존 포스터 S3에 합칠까** — 기각. 성격(공개 vs 민감 운영 데이터), 보관 기간(영구 vs 30~90일 후 삭제), 사고 범위(버킷 하나 사고나면 둘 다 날아감), 모듈 경계(단일 책임 원칙) 전부 분리하는 쪽이 맞다고 판단.

## 1차 범위 vs 2차(나중) 범위

**1차(이번에 같이 구현)**:
1. kube-prometheus-stack + Grafana(클러스터 레벨 지표)
2. Loki + Promtail(로그, S3 백엔드, `03_registry`)
3. Grafana에 CloudWatch 데이터소스(DB 상태)
4. **외부 헬스체크(Blackbox Exporter)** — 오늘 겪은 ExternalDNS/DNS 문제처럼 "클러스터 안은 멀쩡한데 밖에서 접속이 안 되는" 상황을 클러스터 내부 지표만으론 못 잡음. 외부에서 주기적으로 도메인을 직접 호출하는 체크가 있었으면 오늘 문제를 사람이 발견하기 전에 알림이 왔을 것.
5. **ALB 레벨 에러율/응답시간** — CloudWatch(`HTTPCode_Target_5XX_Count`, `TargetResponseTime`)를 Grafana에 연결, 추가 인프라 없이 바로 가능.
6. **배포 시점 annotation** — ArgoCD sync를 Grafana annotation API로 기록, 그래프에 "배포 시점" 세로줄 표시.

**2차(나중, 여유 생기면)**:
7. 비즈니스 지표(예매 성공/실패, 결제 성공률, 대기열 길이) — backend에 Micrometer 커스텀 카운터 추가 필요
8. 비용 모니터링(Kubecost 또는 AWS Cost Explorer 연동) — 오늘 겪은 "고아 리소스가 조용히 과금되는" 상황을 감시
9. 분산 트레이싱(Tempo) — 지금 구조가 단순해서 우선순위 낮음, Grafana LGTM 스택에 자연스럽게 추가 가능

## 트레이데오프 / 남은 리스크

- **미해결**: 알림 채널(Slack vs 이메일) — 팀의 실제 소통 수단 확인 후 결정 필요
- **미해결**: 지표(Prometheus TSDB)용 EBS 볼륨의 AZ 고정 + node affinity 설정 — 설계만 논의, 실제 Terraform 코드는 아직 안 씀
- **미해결**: `03_registry`라는 이름이 "로그/지표 영구 저장소"라는 실제 역할과 안 맞음 — 기능상 문제는 없지만 나중에 팀이 헷갈릴 수 있음(리네임하면 상태 마이그레이션 필요해서 지금은 보류)
- 아직 코드 구현은 전혀 안 됨 — 이 문서는 설계 논의만 정리한 것, 다음 단계는 실제 `02_k8s-addon`/`03_registry` Terraform 코드 작성

## 관련
- [[../troubleshooting/eks-destroy-layer-separation]]
- [[../troubleshooting/cd-helm-chart-deploy-review]]
