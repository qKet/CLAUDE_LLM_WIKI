---
title: Terraform 모듈을 리소스 타입별로 재편 + platform/workload 2-root화
category: decisions
status: 확정
date: 2026-08-06
author: 이채영
tags: [infra, terraform]
---

# Terraform 모듈을 리소스 타입별로 재편 + platform/workload 2-root화

## 배경

기존 구조는 `platform`/`release`/`prod` 3개 디렉토리 root, 모듈은 `network`(vpc+subnet 합침), `eks`, `bastion`, `data`(rds+redis 합침), `storage` 5개(+ helm 설치용 `alb-controller`/`eso`, 지금은 [[../decisions/README|보류 중]])로 나뉘어 있었다.

다른 프로젝트의 참고 구조(개인 예시 스크린샷 + `lunch-12/PAPERPLE-INFRA` 레포)를 보고 "모듈을 리소스 타입 하나당 하나씩(`vpc`/`subnet`/`security_group`/`ec2`/`ecr`/`eks`/`rds`) 나누는 게 유지보수에 더 편하지 않을까"라는 문제 제기가 있었음.

## 결정

- 모듈을 리소스 타입별로 재편: `network`→`vpc`+`subnet`, `data`→`rds`+`redis`, SG는 전부 `security_group`(범용 다중-SG 모듈)로 분리. `eks`/`storage`는 그대로 유지. `ecr` 신규 추가.
- `release`/`prod` 디렉토리 분리는 없애고, **`workload/` 단일 root + terraform workspace(release/prod)**로 대체.
- `platform`(VPC/EKS/bastion/ECR, 공유 싱글턴)은 디렉토리 root 그대로 유지.
- root 파일도 재정리: 모듈 호출 파일들(`vpc.tf`/`subnet.tf`/`eks.tf`/... 각각 따로)을 `main.tf` 하나로, provider 관련 파일들(`provider.tf`/`versions.tf`/`k8s-providers.tf`)을 `providers.tf` 하나로 합침 (PAPERPLE-INFRA 컨벤션과 동일).

자세한 결과 구조는 [[terraform-module-boundaries]], [[terraform-platform-workload-split]] 참고.

## 고려한 대안

- **기존 구조 유지(`network`/`data` 합쳐진 모듈, `release`/`prod` 디렉토리 분리)** — 파일 개수가 적어서(모듈 6개) 단순하지만, "리소스 타입 하나당 모듈 하나"라는 원하는 가독성 목표에는 안 맞음.
- **release/prod까지 포함해서 전부 workspace로 통합** — VPC/EKS까지 workspace마다 중복 생성되어(EKS 컨트롤플레인 비용 2배) 기각. [[terraform-platform-workload-split]]에 상세.
- **`lunch-12/PAPERPLE-INFRA` 구조를 그대로 복붙** — `modules/vpc`가 `modules/subnet`의 출력을(NAT Gateway 배치용), `modules/subnet`이 `modules/vpc`의 출력을(vpc_id) 서로 필요로 하는 **순환 참조**가 실제로 있는 걸 발견해서(레포 raw 파일 직접 확인) 기각. 상세: [[terraform-circular-module-dependency]].

## 트레이드오프 / 남은 리스크

- 파일 개수가 늘어남 (재편 전 모듈 6개 → 9개, `modules/` 하위 총 27개 파일). root 파일은 `main.tf`/`providers.tf`로 합쳐서 상쇄했지만, `modules/`를 펼치면 여전히 많아 보일 수 있음 — "파일 개수"보다 "파일당 내용이 작고 명확한가"가 실제 유지보수성 지표라고 판단해서 감수하기로 함.
- `workload`에 `terraform workspace new release`/`new prod`를 최초 1회 수동으로 만들어줘야 함 — 안 하면 `default` workspace에서 작업하게 되는데, `workspace_guard` precondition이 apply는 막아주지만 이 스텝 자체를 잊으면 첫 시도에서 당황할 수 있음. [[getting-started|온보딩 문서]]에 반영 필요(아직 안 함).
- 아직 `platform`/`workload` 전부 실제로 apply된 적 없는 상태에서 진행함 — 재편 자체는 안전했지만(라이브 리소스 destroy 위험 없음), 실제로 처음 apply할 때 예상 못한 에러가 나올 가능성은 남아있음.

## 관련
- [[terraform-module-boundaries]]
- [[terraform-platform-workload-split]]
- [[terraform-circular-module-dependency]]
- [[2026-08-06-ecr-recreate-vs-import]]
