---
title: 참고 레포(PAPERPLE-INFRA)의 vpc↔subnet 모듈 순환 참조
category: troubleshooting
tags: [infra, terraform]
created: 2026-08-06
updated: 2026-08-10
---

# 참고 레포(PAPERPLE-INFRA)의 vpc↔subnet 모듈 순환 참조

## 증상

Terraform 모듈 재편 시 참고한 `lunch-12/PAPERPLE-INFRA` 레포의 root `main.tf`를 그대로 따라가려다, 다음 패턴을 발견:

```hcl
module "vpc" {
  source = "./modules/vpc"
  public_subnet    = module.subnet.public_subnet   # vpc가 subnet의 출력을 필요로 함
  db_public_subnet = module.subnet.db_public_subnet
}

module "subnet" {
  source = "./modules/subnet"
  vpc_id = module.vpc.vpc_id                        # subnet이 vpc의 출력을 필요로 함
}
```

이대로 가져다 쓰면 `terraform plan`이 `Error: Cycle`로 즉시 실패한다.

## 원인

GitHub raw 파일을 직접 확인한 결과:

- `modules/vpc/main.tf`가 VPC/IGW뿐 아니라 **NAT Gateway까지 만들고 있었음** — NAT는 퍼블릭 서브넷 *안*에 배치해야 하므로 `var.public_subnet`/`var.db_public_subnet`(서브넷 ID)을 입력으로 받아야 했음.
- 반대로 `modules/subnet/main.tf`는 서브넷을 만들려면 `var.vpc_id`가 있어야 함.

즉 "NAT Gateway를 어느 모듈에 둘지"를 `vpc` 모듈로 잘못 배치하면서, `vpc`↔`subnet`이 서로의 출력을 필요로 하는 진짜 순환 의존이 생긴 것. Terraform은 모듈 간 의존성이 항상 단방향 DAG여야 해서 이런 상호참조 자체가 불가능하다.

부가로 그 레포의 `db-nat-table` 라우팅이 `aws_nat_gateway.db`가 아니라 `aws_nat_gateway.main`을 잘못 참조하는 복붙 실수도 같이 발견됨(별개 버그).

## 해결

Qket의 `modules/vpc`/`modules/subnet` 경계는 이 문제를 피하도록 설계함:

- `modules/vpc` = VPC + IGW **만**. subnet 쪽 출력을 전혀 필요로 하지 않음.
- `modules/subnet` = 서브넷 + 라우팅테이블(+연결)만. `vpc_id`/`igw_id`를 입력으로만 받음 — 단방향.

> ⚠️ 모순(2026-08-10 발견): 위에서 "NAT Gateway를 `subnet` 모듈 쪽에 둔다"고 썼었는데, 실제 `Infra/platform/main.tf`를 확인하니 NAT Gateway(`aws_eip.nat`/`aws_nat_gateway.this`)와 라우팅테이블 전부 **`modules/subnet`도 `modules/vpc`도 아닌 root(`platform/main.tf`) 레벨**에 있다. 코드 주석 설명: "vpc·subnet 모듈 둘 다의 출력이 필요해서 root에 둠 — 라우팅은 vpc/subnet 둘 다를 엮는 관심사로 보고 root로 분리". 즉 최종 해법은 "subnet 모듈로 옮기기"가 아니라 "NAT/라우팅을 아예 어느 모듈에도 안 넣고 root로 뺴기"였음. 순환을 피한다는 결론(둘 중 한쪽에만 몰아서 단방향으로 만듦)은 같지만, 실제로 몰아넣은 위치가 이 문서에 적힌 것과 다르다 — **코드가 맞음**, 이 문서가 갱신 안 된 상태로 방치돼 있었다.

## 재발 방지

앞으로 참고 레포/스크린샷 구조를 그대로 옮길 때는, **모듈 간 입력이 서로 맞물리는지(A가 B의 출력을 받고, B도 A의 출력을 받는지)를 먼저 확인**한다. 특히 "서브넷 배치가 필요한 리소스"(NAT Gateway, ALB 등)를 어느 모듈에 둘지가 이런 순환의 단골 원인이다 — 서브넷을 이미 만드는 모듈 쪽에 같이 두면 안전하다.

## 관련
- [[terraform-module-boundaries]]
- [[2026-08-06-terraform-module-restructure]]
