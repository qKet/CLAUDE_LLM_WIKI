---
title: CI 도구로 GitHub Actions 선택 (Jenkins 대신)
category: decisions
status: 확정
date: 2026-08-06
author: 이채영
tags: [ci-cd, github-actions, jenkins]
---

# CI 도구로 GitHub Actions 선택 (Jenkins 대신)

## 배경

레포지토리를 `frontend`/`backend`/`Infra`로 분리하면서 CI/CD를 처음부터 다시 설계하게 됨. 참고한 레퍼런스 아키텍처(다른 프로젝트의 다이어그램)가 Jenkins 기반이었고, 구조는: 서비스 레포 push → Jenkins 파이프라인이 이미지 빌드 → 레지스트리 push → 이미지 태그를 별도 CD 레포에 커밋 → ArgoCD가 CD 레포 변경을 감지해 롤아웃.

이 구조 자체(빌드 → 레지스트리 push → CD 레포 write-back → ArgoCD sync)는 채택하되, **빌드를 실행하는 CI 엔진을 Jenkins로 할지 GitHub Actions로 할지**를 결정해야 했음.

## 결정

**GitHub Actions**를 채택. Jenkins는 쓰지 않음.

파이프라인 구성:
- `backend`/`frontend` 레포 각각에 얇은 워크플로우 파일만 두고, 실제 빌드/ECR push/CD 레포 태그 업데이트 로직은 `qKet/.github` 레포의 재사용 워크플로우(`workflow_call`) 하나에 모아서 중복 관리 부담을 없앰.
- 실행 러너: **GitHub-hosted runner**. `on: push`로 push할 때마다 바로 워크플로우가 도는 기본 방식만 쓰고, self-hosted runner 도입은 지금 보류 (아래 "보류된 하위 결정" 참고).
- AWS 인증은 OIDC role assume (정적 키 미사용).
- CD 레포(`qKet/CD`)는 Terraform/Infra와 완전히 분리된 순수 GitOps 매니페스트 전용 레포.

## 채택 이유

- **이미 조직/코드가 전부 GitHub에 있음** — `qKet` organization, 모든 레포(`frontend`/`backend`/`Infra`/`docs`/`CLAUDE_LLM_WIKI`)가 GitHub. 다른 VCS나 외부 CI 플랫폼으로 옮길 이유가 없음.
- **필요한 준비물이 이미 갖춰져 있었음** — 예전 모노레포(`Q_Ket_old`)에서 GitHub Actions 기준 ECR IAM 정책(`docs/github-actions-ecr-policy.json`)과 실제 동작하던 워크플로우(`build-deploy-release.yml`, `terraform-*.yml`)를 이미 한 번 만들어봤음. 도구를 바꾸면 이 자산이 전부 무효화됨.
- **서버 운영 부담이 없음** — GitHub-hosted runner를 쓰면 빌드 서버를 직접 프로비저닝·패치·업그레이드할 필요가 없음. Jenkins는 이 부담이 그대로 팀(4명)에게 옴.
- **Jenkins 대비 실질적 기능 차이가 없었음** — 논의 과정에서 "Jenkins가 러너 소유·push 즉시 감지에 유리하다"는 초기 인상이 있었지만, 확인해보니 GitHub Actions도 self-hosted runner 지원 + `on: push` 기본 반응성 + 재사용 워크플로우로 동일한 걸 제공함 (아래 "고려한 대안" 참고). 즉 Jenkins를 골라도 얻는 추가 이득이 없는데 서버 운영 비용만 더 지는 셈이라 GitHub Actions가 합리적.
- **작은 팀 규모에 맞음** — 팀원 4명, Owner 1명 수준 규모에서 별도 CI 서버를 운영·인수인계하는 건 과한 오버헤드.

## Jenkins 대비 구조적 장점

위 "채택 이유"가 **우리 상황이라서** 고른 이유였다면, 아래는 상황과 무관하게 항상 성립하는 GitHub Actions의 구조적 장점.

- **서버 자체가 없음(완전 관리형)** — Jenkins는 서버(EC2든 EKS 안이든)를 팀이 직접 띄우고 계속 살아있게 유지해야 함. OS 패치, Jenkins 코어/플러그인 업그레이드, 디스크 용량 관리, 플러그인 호환성 깨짐 대응까지 전부 팀 몫. GitHub Actions는 이 전부가 GitHub 책임.
- **단일 장애점이 없음** — Jenkins 서버가 죽으면(디스크 풀, 크래시, 재시작 필요) 모든 팀의 CI가 동시에 멈춤. GitHub Actions는 팀이 지켜야 할 서버 자체가 없어서 이 리스크가 원천적으로 없음.
- **GitHub 기능과 훨씬 긴밀하게 통합됨** — PR에 상태 체크/코드 라인별 주석 자동 표시, Checks API, `GITHUB_TOKEN`이 워크플로우 실행마다 자동 스코프됨, `environments`의 Required reviewers(이미 `terraform-platform.yml`에서 쓰던 승인 게이트) 같은 게 다 네이티브 기능. Jenkins도 플러그인으로 비슷하게 흉내는 내지만 항상 한 겹 더 얹은 느낌이고 설정이 GitHub 상태와 어긋날 여지가 있음.
- **마켓플레이스 품질/최신성** — `aws-actions/configure-aws-credentials`처럼 AWS 등 벤더가 직접 만들어서 유지보수하는 action이 많음. Jenkins 플러그인은 커뮤니티 유지보수라 방치되는 경우가 잦음.
- **보안 패치 부담이 GitHub로 넘어감** — Jenkins는 코어/플러그인에 CVE가 꽤 자주 나와서 팀이 계속 추적·패치해야 함. GitHub Actions 러너/플랫폼은 GitHub이 패치.
- **팀 온보딩이 더 쉬움** — 모든 팀원이 이미 GitHub 계정/권한 체계 안에 있어서, Jenkins 로그인·RBAC를 별도로 관리할 필요가 없음.

즉 "Jenkins로 가도 기능적으로 똑같이 만들 수는 있다"는 맞지만, **그 기능을 계속 살아있게 유지하는 책임을 GitHub이 대신 지느냐 팀이 직접 지느냐**가 핵심 차이.

## 고려한 대안: Jenkins (EKS 클러스터 내부 or 별도 EC2)

논의 과정에서 나온 오해와 실제 비교:

- **"Jenkins는 러너를 직접 둘 수 있어서 유리하다"는 초기 추정은 틀림** — GitHub Actions도 self-hosted runner를 지원해서, "직접 러너를 두고 싶다"는 니즈 자체는 두 도구 모두 채울 수 있음. 러너 소유 여부는 둘의 진짜 차이가 아니었음.
- **"push 감지"도 Jenkins만의 강점이 아님** — GitHub Actions는 `on: push`만으로 즉시 반응(추가 설정 없음). Jenkins는 GitHub 레포에 웹훅을 직접 등록해야 같은 반응성을 얻음 (Jenkins의 "GitHub Organization" job이 이 등록을 자동화해주긴 하지만, 내부적으론 결국 웹훅 방식).
- **레포별 파이프라인 설정 중복 문제도 두 도구 모두 동일하게 존재** — GitHub Actions는 재사용 워크플로우(`workflow_call`)로, Jenkins는 Shared Library로 해결. 이 축에서도 우열 없음.

Jenkins가 실제로 우위인 경우(참고용, 지금은 해당 없음):
- 빌드량이 매우 많아 hosted runner 분당 과금이 자체 서버 고정비보다 비쌀 때
- GitHub이 아닌 다른 VCS까지 하나의 CI로 통합해야 할 때
- 폐쇄망/온프레미스라 외부 SaaS(GitHub Actions) 자체를 못 쓸 때
- 이미 Jenkins 운영 노하우/플러그인 의존이 있는 팀일 때

## 보류된 하위 결정: self-hosted runner

지금은 GitHub-hosted runner + `on: push` 기본 방식만 쓰기로 하고, self-hosted runner 도입은 결정하지 않고 **보류**함. 나중에 아래 조건이 생기면 재검토:

- 빌드량이 늘어나 hosted runner 분당 과금이 부담될 때
- 빌드 중 사설망(VPC) 안 리소스에 접근해야 할 일이 생길 때
- 빌드 캐시/속도를 위해 전용 하드웨어가 필요할 때

## 트레이드오프 / 남은 리스크

- GitHub Actions hosted runner는 분당 과금이라, **빌드량이 크게 늘어나면** 자체 러너(self-hosted) 전환을 재검토해야 할 수 있음 — 다만 이건 Jenkins로 갔어도 똑같이 서버 스케일링 문제로 마주쳤을 이슈.
- 지금 조직이 전부 GitHub에 있다는 전제가 바뀌면(예: 다른 VCS 도입) 이 결정도 재검토 대상.
- Jenkins 서버를 안 띄우기로 했으므로, Jenkins 플러그인 생태계(예: 특정 니치 통합)가 필요해지면 GitHub Actions Marketplace에서 대체재를 찾거나 커스텀 action을 직접 만들어야 함.

## 관련
- [[../onboarding/getting-started]]
- (참고, 다른 레포) `docs/ci-cd-cross-repo-auth.md` — CD 레포에 write-back할 때 인증 방식(PAT vs GitHub App) 비교


