---
title: 이메일 발송 인프라 (SQS + Lambda + SES)
category: architecture
tags: [sqs, lambda, ses, messaging, terraform]
created: 2026-08-12
updated: 2026-08-12
---

# 이메일 발송 인프라 (SQS + Lambda + SES)

## 구조 / 요약

이메일 인증번호(현재) / 예매완료·결제내역(예정) 발송을 위한 큐+함수 조합. backend가 SQS에 메시지를 넣으면 Lambda가 트리거되어 SES로 실제 발송한다 — backend가 SES를 직접 호출하지 않고 큐를 거치는 이유는 SES 호출 실패/지연이 결제·예매 같은 핵심 흐름의 응답 시간에 영향을 안 주게 하기 위함(비동기 분리). **왜 SQS/Lambda/SES라는 조합 자체를 골랐는지(SNS·EventBridge Scheduler와 비교)**는 별도 문서 [[../decisions/2026-08-12-sqs-lambda-ses-notification-pipeline]] 참고 — 이 문서는 "지금 이렇게 되어있다"만 다룬다.

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

## SES 발신 도메인(`jun979.click`) 인증 현황

| 항목 | 상태 |
|---|---|
| DKIM | ✅ CNAME 3개 등록 완료 — `03_registry/ses.tf`의 `aws_ses_domain_dkim`/`aws_route53_record.ses_dkim`이 관리. 발신 서명 인증 |
| DMARC | ✅ TXT 등록 완료 — 현재 모니터링 모드(`p=none`), Terraform 코드 밖에서(수동으로) 설정된 것으로 보임 — 아직 이 레코드를 관리하는 `.tf` 리소스가 없음 |
| SPF | ⚠️ **미등록** — 루트 도메인 TXT에 `v=spf1 include:amazonses.com ~all` 추가 필요. 미해결 액션 아이템 |
| MAIL FROM | 선택 사항 — 없어도 발송 자체는 가능, 추후 정렬률(alignment) 개선용 |
| SES 프로덕션 액세스 | ✅ 승인 완료 — 샌드박스 해제, 검증 안 된 임의 수신자에게도 발송 가능 |

## 의도적으로 어색하거나 특이한 지점

- Lambda 코드가 지금 실제로 처리하는 메시지 스펙은 `{ email, code }`(이메일 인증번호)뿐이다 — 예매확정/취소 알림([[../decisions/2026-08-12-sqs-lambda-ses-notification-pipeline]]이 다루는 유스케이스)은 아직 코드로 연결 안 됨. 위 "확장하는 방법"대로 `type` 필드 분기를 추가하면 같은 큐/Lambda로 처리 가능하지만, 실제 반영은 아직 안 됐다.
- DMARC 레코드가 Terraform 밖에서 설정된 것으로 보여 `terraform plan`에 안 잡힌다 — 나중에 이 도메인의 다른 레코드를 Terraform으로 관리하려 할 때 SES DKIM 때 겪은 것과 같은 "already exists" 충돌([[../troubleshooting/ses-dkim-preexisting-records-import]])이 재현될 수 있다는 걸 미리 인지해둘 것.

## 관련
- [[../decisions/2026-08-12-messaging-infra-placement]]
- [[../decisions/2026-08-12-sqs-lambda-ses-notification-pipeline]]
- [[../troubleshooting/ses-dkim-preexisting-records-import]]
- [[../troubleshooting/lambda-env-var-and-runtime-version-gotchas]]
- [[terraform-platform-workload-split]]
