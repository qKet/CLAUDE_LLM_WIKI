---
title: 이메일 발송 인프라 (SQS + Lambda + SES)
category: architecture
tags: [sqs, lambda, ses, messaging, terraform]
created: 2026-08-12
updated: 2026-08-12
---

# 이메일 발송 인프라 (SQS + Lambda + SES)

## 구조 / 요약

이메일 인증번호(현재) / 예매완료·결제내역(예정) 발송을 위한 큐+함수 조합. backend가 SQS에 메시지를 넣으면 Lambda가 트리거되어 SES로 실제 발송한다 — backend가 SES를 직접 호출하지 않고 큐를 거치는 이유는 SES 호출 실패/지연이 결제·예매 같은 핵심 흐름의 응답 시간에 영향을 안 주게 하기 위함(비동기 분리).

```
backend → SQS(email_verification) → Lambda(email_verification) → SES → 실제 메일함
                ↓ (3회 실패)
              DLQ(email_verification_dlq, 14일 보관)
```

리소스가 **세 곳에 나뉘어 있다**:

| 무엇 | 어디 | 왜 |
|---|---|---|
| SQS 큐 + DLQ, Lambda 함수, Lambda IAM Role | `modules/messaging` (`04_data`가 release/prod 각각 호출) | 인증 발송량이 환경별로 분리돼야 함(테스트 발송이 실제 서비스랑 안 섞이게) — `team5-qket-email-verification-{release,prod}` 이름으로 각자 존재 |
| SES 도메인 인증(identity + DKIM 3개 CNAME) | `03_registry` (`ses.tf`) | 도메인 하나에 대한 SES 인증은 **계정당 한 번만** 해야 하는데, `04_data`는 release/prod 두 워크스페이스로 두 번 apply된다 — `modules/messaging`에 두면 같은 도메인을 두 번 인증하려다 충돌한다(실제로 이 실수를 했다가 겪은 사고는 [[../troubleshooting/ses-dkim-preexisting-records-import]]). `03_registry`는 이미 "공유·환경 구분 없음·절대 안 지움" 성격(ECR/OIDC)이라 SES 도메인 인증도 같은 카테고리로 배치 |
| Lambda 코드 (`lambda-src/index.mjs`) | `modules/messaging` 내부 | `data "archive_file"`가 이 폴더를 zip으로 압축해서 그대로 Lambda에 올림 — 아래 "Lambda 배포 방식" 참고 |

## Lambda 배포 방식 — CI 없이 `terraform apply` 하나로 끝남

`modules/messaging/lambda.tf`의 `data "archive_file" "email_verification"`이 `terraform apply` 실행 시점마다 `lambda-src/` 폴더 전체를 zip으로 재압축하고, `aws_lambda_function`이 `source_code_hash`(zip 내용 기반 해시)로 변경을 감지해서 자동 재배포한다. 즉 backend/frontend처럼 별도 CI 빌드→ECR push 파이프라인이 필요 없다 — **`lambda-src/index.mjs`를 고치고 `04_data`를 apply하면 그게 곧 배포**다. 자세한 배치 이유는 [[../decisions/2026-08-12-messaging-infra-placement]].

외부 npm 패키지가 필요해지면 `node_modules`도 `lambda-src/` 안에 같이 있어야 zip에 포함된다 — 단, `@aws-sdk/client-sesv2` 같은 AWS SDK v3 클라이언트는 Lambda Node.js 런타임에 기본 내장돼 있어서 별도 설치 없이 바로 동작한다.

## 권한 / 트리거

- `aws_lambda_event_source_mapping`(SQS→Lambda, `batch_size = 1`, 폴링 방식) — 메시지 하나씩 즉시 처리
- Lambda IAM Role은 딱 필요한 만큼만 열려 있음:
  - SQS: `ReceiveMessage`/`DeleteMessage`/`GetQueueAttributes` — **이 큐 하나(ARN)로만** 범위 제한
  - SES: `SendEmail`/`SendRawEmail` — **이 도메인 identity(`jun979.click`) ARN으로만** 범위 제한. 이 ARN은 계정/리전/도메인명으로 완전히 결정되는 값(랜덤 접미사 없음)이라 `data.aws_caller_identity`/`data.aws_region`으로 직접 계산 — `03_registry`의 실제 리소스를 `terraform_remote_state`로 참조할 필요가 없음(cross-root 결합을 피함)
- DLQ로 재시도 3회 실패한 메시지를 격리(`redrive_policy`, `maxReceiveCount = 3`) — 무한 재시도로 쌓이는 것 방지, 14일간 원인 조사 가능

## 확장하는 방법 (예: 예매완료/결제내역 이메일 추가)

새 SQS 큐·Lambda를 또 만들 필요 없이, **기존 큐/Lambda를 재사용하고 메시지에 `type` 필드로 분기**하는 방향을 권장(인프라 리소스 추가 없이 `index.mjs` 코드 수정만으로 끝남). SES 발송 권한도 이미 도메인 단위로 열려 있어서 이메일 종류가 늘어나는 것과 무관하다.

## 관련
- [[../decisions/2026-08-12-messaging-infra-placement]]
- [[../troubleshooting/ses-dkim-preexisting-records-import]]
- [[../troubleshooting/lambda-env-var-and-runtime-version-gotchas]]
- [[terraform-platform-workload-split]]
