---
title: 프론트 CI 빌드 시점 `NEXT_PUBLIC_TOSS_CLIENT_KEY`를 GitHub Secrets 대신 Secrets Manager에서 직접 fetch
category: decisions
status: 확정
date: 2026-08-11
author: 이채영
tags: [ci, secrets-manager, oidc, frontend, toss]
---

# 프론트 CI 빌드 시점 `NEXT_PUBLIC_TOSS_CLIENT_KEY`를 GitHub Secrets 대신 Secrets Manager에서 직접 fetch

## 배경

`NEXT_PUBLIC_*` 접두사가 붙은 Next.js 환경변수는 **빌드 타임에 번들에 박힌다** — 런타임에 K8s Secret으로 주입해봐야 이미 빌드된 정적 자산에는 반영이 안 되므로, CI가 빌드하는 시점에 이 값을 알고 있어야 한다. `TOSS_CLIENT_KEY`는 [[2026-08-11-external-api-secrets-manager]]로 Secrets Manager에 이미 저장돼 있는데, 이 값을 프론트 CI(`frontend/.github/workflows/CI-release.yml`)로 어떻게 가져올지 결정이 필요했다.

## 결정

**GitHub Secrets에 별도로 값을 복제하지 않고, CI가 이미 갖고 있는 AWS OIDC 인증으로 Secrets Manager에서 직접 읽는다.**

- "Configure AWS credentials (OIDC)" 스텝을 빌드보다 먼저 실행되도록 순서 조정
- 새 스텝 "Fetch Toss client key from Secrets Manager":
  ```bash
  KEY=$(aws secretsmanager get-secret-value \
    --secret-id team5-qket-external-api-release \
    --query SecretString --output text | jq -r .TOSS_CLIENT_KEY)
  echo "::add-mask::$KEY"
  echo "key=$KEY" >> "$GITHUB_OUTPUT"
  ```
- `Install & build` 스텝에 `env: NEXT_PUBLIC_TOSS_CLIENT_KEY: ${{ steps.toss.outputs.key }}` 로 주입
- `::add-mask::`로 Action 로그에 값이 그대로 찍히는 것 방지
- IAM 쪽은 `modules/github-actions-oidc`에 `aws_iam_role_policy "frontend_secrets_read"`를 새로 추가 — `arn:aws:secretsmanager:REGION:ACCOUNT:secret:team5-qket-external-api-*` 이름 접두사 와일드카드로 범위를 좁히고, frontend CI Role에만 붙임(backend CI Role엔 안 붙임 — backend는 이 값이 필요 없음)

## 고려한 대안

- **GitHub Secrets에 값을 복사해서 저장** — 기각. 사용자가 명시적으로 거부("깃 씨크릿은 쓰기싫은데"). 값의 원본(source of truth)이 Secrets Manager 하나로 유지되고, 로테이션·감사·삭제가 모두 한 곳에서 일어나는 걸 선호 — GitHub Secrets에 복제본을 두면 두 저장소의 값이 어긋날 위험(하나만 갱신하고 잊어버리는 것)이 생긴다.
- **ESO로 K8s Secret에 동기화한 뒤 빌드 시점에 K8s에서 읽기** — 불가능. CI 빌드는 클러스터 바깥(GitHub-hosted runner)에서 일어나고, K8s Secret은 클러스터 안에만 존재. 애초에 `NEXT_PUBLIC_*` 특성상 "런타임에 클러스터에서 읽기"라는 개념 자체가 안 맞음(빌드 타임에 필요).

## 트레이드오프 / 남은 리스크

- **하드코딩된 `-release` 접미사**: 지금 CI는 `team5-qket-external-api-release` 시크릿 이름을 고정으로 참조 — prod CI 파이프라인이 아직 없어서 임시로 이렇게 둠. prod CI를 만들 때 워크스페이스/환경별로 시크릿 이름을 분기하도록 반드시 갱신해야 함.
- OIDC Role의 정책이 시크릿 **이름 접두사** 와일드카드로 열려 있어서, `team5-qket-external-api-*`로 시작하는 어떤 시크릿이든 frontend CI가 읽을 수 있다 — 지금은 release/prod 두 개뿐이라 실질적 리스크는 낮지만, 이 접두사 아래 다른 목적의 시크릿을 추가할 계획이 있다면 재검토 필요.

## 관련
- [[2026-08-11-external-api-secrets-manager]]
- [[../architecture/terraform-remote-state]]
