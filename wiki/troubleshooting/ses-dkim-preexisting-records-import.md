---
title: "terraform apply on 03_registry — SES DKIM CNAME 레코드가 이미 존재해서 실패"
category: troubleshooting
tags: [terraform, ses, route53, dkim, import]
created: 2026-08-12
updated: 2026-08-12
---

# `terraform apply` on `03_registry` — SES DKIM CNAME 레코드가 이미 존재해서 실패

## 증상

`03_registry`(`ses.tf` 신설 직후, [[../decisions/2026-08-12-messaging-infra-placement]])에서 첫 `terraform apply`를 돌렸는데, DKIM용 Route53 CNAME 레코드 3개 전부 다음 에러로 실패:

```
InvalidChangeBatch: [Tried to create resource record set
[name='<dkim-token>._domainkey.jun979.click.', type='CNAME']
but it already exists]
```

## 원인

`terraform state list`로 확인해보니 `aws_ses_domain_identity`/`aws_ses_domain_identity_verification`/`aws_ses_domain_dkim`은 이미 정상적으로 state에 들어가 있었고(멱등이라 Terraform이 기존 걸 그대로 재확인하고 넘어감 — DKIM 토큰도 매번 같은 값이 나옴), **실패한 건 딱 3개, 그 DKIM 토큰을 가리키는 Route53 CNAME 레코드 CREATE뿐**이었다.

즉 SES 도메인 인증 자체가 이 Terraform 코드가 생기기 전에 **이미 되어 있었다** — 팀이 이전에 SES 프로덕션 액세스를 신청해뒀다는 사실(`aws sesv2 get-account`로 확인한 `ProductionAccessEnabled: true`, `SentLast24Hours: 5`)과 일치한다. 아마 콘솔에서 수동으로, 또는 이 Terraform 코드와 무관한 경로로 먼저 인증해뒀던 것. Terraform은 자기 state에 없는 리소스이므로 "새로 만들어야 한다"고 판단해서 CREATE를 시도했는데, AWS 쪽에는 동일한 이름의 레코드가 이미 있어서 충돌.

## 해결

`terraform state show aws_ses_domain_dkim.this`로 정확한 토큰 순서를 확인한 뒤, `terraform import`로 기존 Route53 레코드 3개를 각각 state에 편입:

```bash
terraform import 'aws_route53_record.ses_dkim[0]' \
  Z0111999JD2RHOSHTM8A_<token0>._domainkey.jun979.click._CNAME
terraform import 'aws_route53_record.ses_dkim[1]' \
  Z0111999JD2RHOSHTM8A_<token1>._domainkey.jun979.click._CNAME
terraform import 'aws_route53_record.ses_dkim[2]' \
  Z0111999JD2RHOSHTM8A_<token2>._domainkey.jun979.click._CNAME
```

import ID 형식은 `{zone_id}_{record_name}_{record_type}`. import 후 `terraform plan`을 돌려보니 실제 남은 diff는 TTL 값 차이(기존 레코드 1800초 → 코드에는 600초)뿐이었고, `terraform apply`로 TTL만 갱신하고 나머지는 그대로 — `0 added, 3 changed, 0 destroyed`로 완전히 정리됨.

## 재발 방지

- **AWS 리소스가 Terraform 코드보다 먼저 존재할 수 있는 상황**(콘솔 수동 작업, 다른 사람이 먼저 설정, 팀 신청 절차가 코드화 전에 이미 끝난 경우 등)에서는 `apply` 에러가 "already exists" 계열이면 곧바로 `terraform import`를 먼저 검토한다 — 삭제 후 재생성은 SES처럼 도메인 인증에 시간이 걸리거나 부작용이 있는 리소스에는 특히 위험하다.
- `count`/`for_each`로 만들어지는 리소스(`aws_route53_record.ses_dkim[N]`)를 import할 때는 인덱스 순서가 실제 값 순서와 일치하는지 `terraform state show`로 먼저 확인 — 순서를 잘못 맞추면 import는 성공해도 이후 plan에서 엉뚱한 diff가 나올 수 있다.

## 관련
- [[../decisions/2026-08-12-messaging-infra-placement]]
- [[../architecture/messaging-infrastructure]]
