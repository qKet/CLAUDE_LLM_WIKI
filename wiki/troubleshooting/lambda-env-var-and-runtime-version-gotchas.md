---
title: Lambda 삽질 둘 — 예약 환경변수 AWS_REGION, nodejs24.x가 provider에서 안 먹힘
category: troubleshooting
tags: [lambda, terraform, nodejs, provider]
created: 2026-08-12
updated: 2026-08-12
---

# Lambda 삽질 둘 — 예약 환경변수 `AWS_REGION`, `nodejs24.x`가 provider에서 안 먹힘

메시징 Lambda([[../architecture/messaging-infrastructure]])를 처음 배포하면서 겪은 작은 삽질 두 개. 각자 원인이 달라서 하나로 묶어 기록.

## 1. `AWS_REGION`을 environment variable로 직접 설정하면 `CreateFunction`이 거부됨

### 증상
```
InvalidParameterValueException: Lambda was unable to configure your environment
variables because the environment variables you have provided contains reserved
keys that are currently not supported for modification. Reserved keys used in
this request: AWS_REGION
```

### 원인
`AWS_REGION`은 Lambda가 **런타임이 자동으로 주입하는 예약 환경변수**라 사람이 직접 설정할 수 없다. `aws_lambda_function.email_verification`의 `environment.variables`에 `AWS_REGION = var.aws_region`을 넣었던 게 원인.

### 해결
`environment` 블록에서 `AWS_REGION`을 아예 뺐다 — Lambda 코드가 `process.env.AWS_REGION`을 그대로 읽으므로(SES 클라이언트 초기화에 씀), 런타임이 자동으로 채워주는 값을 그대로 쓰면 된다. `var.aws_region`은 이 모듈의 다른 곳(`iam.tf`의 SES ARN 계산)에서는 여전히 필요해서 변수 자체는 유지.

## 2. `runtime = "nodejs24.x"`가 provider에서 "expected runtime to be one of [...]" 에러

### 증상
```
Error: expected runtime to be one of ["...", "nodejs20.x", "provided.al2023",
"python3.12", "java21", "python3.13", "nodejs22.x"], got nodejs24.x
```

### 원인
`nodejs24.x`는 AWS Lambda 자체에는 2025년 11월부터 정식 지원되는 런타임이지만(2028년 4월까지 지원되는 LTS), **`terraform-provider-aws`가 이 값을 정식으로 허용 목록에 넣은 건 v6.21.0부터**다. 이 프로젝트는 전 root(`01_infrastructure`~`04_data`)가 `hashicorp/aws` provider를 `~> 5.0`으로 고정하고 있고, 실제 설치된 버전은 `5.100.0` — client-side validation 단계에서 바로 막힘(AWS API까지 가지도 못함).

### 해결 방향 (진행 중)
AWS provider를 5.x → 6.x(v6.21.0+)로 메이저 업그레이드해야 한다 — 단, 이건 `modules/messaging`만의 문제가 아니라 **4개 root 전부가 같은 버전 제약을 공유**하고 있어서, 하나만 올리는 게 아니라 전체를 breaking change 검토와 함께 올려야 한다. 급하지 않다면 `nodejs22.x`(2027년 4월까지 지원되는 LTS)를 그대로 쓰는 것도 당장은 문제 없는 선택지.

> 이 항목은 실제로 v6 업그레이드를 완료한 뒤에 결과(어떤 breaking change를 만났는지, 몇 버전으로 정착했는지)로 갱신 필요 — 아직 진행 전.

## 재발 방지

- Lambda `environment.variables`에 값을 넣기 전에 [AWS 예약 환경변수 목록](https://docs.aws.amazon.com/lambda/latest/dg/configuration-envvars.html)(`AWS_REGION`, `AWS_ACCESS_KEY_ID`, `_HANDLER` 등)과 겹치지 않는지 먼저 확인한다.
- 새로 나온 런타임/버전을 쓰려고 할 때는 AWS 자체 지원 여부뿐 아니라, **쓰고 있는 Terraform provider 버전이 그 값을 허용 목록에 포함하는지**도 같이 확인한다 — 이 프로젝트처럼 provider를 메이저 아래(`~> 5.0`)로 묶어두면 AWS가 지원을 시작해도 몇 달씩 못 쓸 수 있다.

## 관련
- [[../architecture/messaging-infrastructure]]
- [[../architecture/terraform-platform-workload-split]]
