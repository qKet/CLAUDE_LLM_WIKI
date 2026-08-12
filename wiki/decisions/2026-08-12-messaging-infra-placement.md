---
title: 이메일 발송 인프라(SQS+Lambda+SES) — 새 root 대신 04_data 재사용, 로컬 zip 배포
category: decisions
status: 확정
date: 2026-08-12
author: 이채영
tags: [terraform, sqs, lambda, ses, messaging]
---

# 이메일 발송 인프라(SQS+Lambda+SES) — 새 root 대신 `04_data` 재사용, 로컬 zip 배포

## 배경

팀원이 SES/SQS/Lambda 기반 이메일 인증번호 발송 인프라를 제안(비용 분석 포함). 구조 자체(SQS→Lambda→SES)는 대부분 동의했고, 이걸 기존 4-root 구조(`01_infrastructure`~`04_data`, [[../architecture/terraform-platform-workload-split]])에 어떻게 배치할지와 Lambda 코드를 어떻게 배포할지를 결정해야 했다.

## 결정

1. **새 root(`05_messaging`) 대신 기존 `04_data`에 `module "messaging"`으로 접붙인다.** SQS/Lambda는 RDS/Redis/S3와 마찬가지로 release/prod마다 따로 있어야 하는 "환경별 데이터/워크로드성 리소스"라 `04_data`의 성격과 정확히 일치 — root를 새로 쪼갤 만큼 독립적인 lifecycle이 아니다.
2. **단, SES 도메인 인증(identity+DKIM)만은 `03_registry`로 뺀다.** 도메인 인증은 계정당 1회만 해야 하는데 `04_data`는 release/prod 두 워크스페이스로 두 번 apply되므로, `modules/messaging` 안에 두면 충돌한다(자세한 사고는 [[../troubleshooting/ses-dkim-preexisting-records-import]]). `03_registry`는 이미 ECR/OIDC로 "공유·환경 구분 없음" 카테고리를 갖고 있어서 그대로 재사용.
3. **Lambda 코드 배포는 CI/S3 업로드가 아니라 Terraform `archive_file`로 로컬 zip 패키징한다.** 코드가 순수 Node.js(외부 빌드 스텝 없음)라, `terraform apply` 한 번으로 압축부터 배포까지 끝나는 게 가장 단순하다.

## 고려한 대안

- **새 root(`05_messaging`) 신설** — 기각. 4-root 구조가 이미 "AWS 순수 인프라 / K8s addon / 공유·불변 / 환경별 데이터"라는 명확한 분류 기준을 갖고 있는데, 메시징 인프라는 이 중 어디에도 새 카테고리를 만들 필요 없이 "환경별 데이터"에 바로 들어맞는다. root를 늘릴수록 apply 순서·backend 설정·팀원 온보딩 비용이 같이 늘어난다.
- **Lambda 코드를 CI가 빌드해서 S3에 업로드하고, Terraform은 그 S3 키만 참조(`s3_bucket`/`s3_key`)** — 처음엔 이 방식을 권장했으나(빌드 스텝이 필요한 복잡한 코드를 상정), 실제 코드가 외부 의존성 설치나 트랜스파일 없이 그대로 동작하는 순수 Node.js라는 게 확인되면서 기각. CI 파이프라인 하나를 통째로 새로 만드는 비용 대비 얻는 이득이 없음 — 나중에 코드가 복잡해져서 빌드 스텝이 필요해지면 이 결정을 재검토해야 한다.

## 트레이드오프 / 남은 리스크

- `archive_file` 방식은 **빌드 스텝이 필요 없는 코드에서만** 성립한다 — TypeScript 컴파일, 번들링, 외부 패키지 설치가 필요해지는 순간 이 결정을 재검토해야 함(그때는 CI-built S3 zip 방식이 다시 유력해짐).
- 콘솔에서 이미 수동으로 만들어져 있던 `qket-email-verification-lambda`(Terraform 관리 밖)가 새 Terraform 관리 함수(`team5-qket-email-verification-release`)와 별개로 남아있음 — 새 함수가 실제로 잘 동작하는 게 확인되면 정리(삭제) 필요.

## 관련
- [[../architecture/messaging-infrastructure]]
- [[../architecture/terraform-platform-workload-split]]
- [[../troubleshooting/ses-dkim-preexisting-records-import]]
