---
title: 기존 ECR 저장소를 import 대신 삭제 후 Terraform으로 재생성
category: decisions
status: 확정
date: 2026-08-06
author: 이채영
tags: [infra, terraform, ecr]
---

# 기존 ECR 저장소를 import 대신 삭제 후 Terraform으로 재생성

## 배경

`team5/ecr/qket` ECR 저장소가 예전 모노레포 CI로 수동(비-Terraform)으로 만들어져 있었고, `modules/ecr`을 새로 만들면서 이걸 Terraform 관리 대상으로 편입시켜야 했다. 확인해보니 이 저장소엔 **이미지가 140개 이상** 들어있었음(`backend-*`/`frontend-*` 태그로 커밋별 빌드 기록이 누적된 상태).

## 결정

**`terraform import`로 기존 저장소를 편입시키지 않고, 저장소를 완전히 삭제(이미지 140개+ 전부 삭제)한 뒤 Terraform이 처음부터 새로 생성하게 함.**

## 고려한 대안

- **`terraform import`** — 기존 저장소/이미지를 그대로 두고 Terraform 관리 대상으로만 편입. 이미지 히스토리 보존, 재생성/다운타임 없음. **이게 일반적으로 권장되는 방법**이라 먼저 제안했음.

## 트레이드오프 / 남은 리스크

- 예전 모노레포 시절 이미지 빌드 히스토리(어떤 커밋이 어떤 이미지로 빌드됐는지 추적 가능했던 기록)가 전부 사라짐. 롤백이 필요한 상황에서 예전 이미지로 되돌릴 수 없음.
- 새 ECR 저장소는 [[terraform-module-boundaries|modules/ecr]]의 lifecycle policy(untagged 이미지 7일 후 자동 만료)가 처음부터 적용된 상태로 시작 — 이번 삭제를 계기로 관리되지 않는 이미지가 쌓이는 문제 자체는 재발 방지됨.
- 삭제는 `aws ecr delete-repository --force`로 실행 완료, 되돌릴 수 없음 (2026-08-06 진행 전 이미지 목록 조회로 개수 확인 후 사용자 명시적 확인 받고 진행).

## 관련
- [[terraform-module-boundaries]]
- [[2026-08-06-terraform-module-restructure]]
