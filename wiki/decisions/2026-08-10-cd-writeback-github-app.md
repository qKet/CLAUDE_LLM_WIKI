---
status: 확정
date: 2026-08-10
author: 이채영
---

# CD 레포 write-back 인증 방식으로 GitHub App 선택 (PAT 대신)

## 배경

backend/frontend CI가 이미지를 ECR에 push한 뒤, 실제 배포는 ArgoCD가 `qKet/CD` 레포를 보고 하기 때문에, CI가 그 이미지 태그를 `CD` 레포의 `helm/values.yaml`에 반영(write-back)해야 한다. GitHub Actions 기본 `GITHUB_TOKEN`은 자기 레포(backend/frontend)에만 유효해서 다른 레포(`CD`)를 건드릴 권한이 없다 — 그래서 별도 인증 수단이 필요했다.

`docs/cluadeDocs/ci-cd-cross-repo-auth.md`에 이미 두 선택지(Fine-grained PAT / GitHub App)의 장단점을 비교해뒀었는데, 그때는 결론을 안 내고 비교만 해둔 상태였다.

## 결정

**GitHub App**으로 결정 — `qket-ci-bot`(qKet 조직 소유), `Contents: Read and write` 권한만, `qKet/CD` 레포 하나에만 설치.

CI 워크플로우(`backend`/`frontend`의 `CI-release.yml`)는 `actions/create-github-app-token@v1`로 **실행할 때마다 ~1시간짜리 설치 토큰을 새로 발급**받아서 그 토큰으로 `qKet/CD`를 checkout → `helm/values.yaml`의 image 값 갱신 → commit/push한다.

## 고려한 대안: Fine-grained PAT

레포 하나(`CD`)로 범위를 좁히고 만료 기한을 설정하면 Fine-grained PAT도 "많이" 위험한 선택은 아니었다 — Classic PAT(계정 전체 권한)와 달리 애초에 레포/권한 단위로 좁힐 수 있게 나온 기능이라서. 실제로 이 판단까지 사용자와 같이 짚어봤다.

## 트레이드오프 / 왜 그래도 GitHub App인가

PAT의 "기술적 위험도"보다 **"장기 자격증명을 시크릿에 넣어두는 것 자체가 싫다"**는 원칙적인 이유가 결정적이었다. 이건 이 프로젝트가 처음부터 일관되게 유지해온 방향이기도 하다 — AWS 인증도 고정 액세스 키 대신 OIDC로 갔던 것([[../decisions/2026-08-06-ci-tool-github-actions]] 및 backend `CI-release.yml`의 OIDC 관련 주석 참고)과 같은 맥락. GitHub App은 시크릿에 실제로 저장되는 게 private key뿐이고, API 호출에 바로 쓸 수 있는 토큰은 매번 새로 발급되는 단명 토큰이라 이 원칙에 맞는다.

초기 설정이 PAT보다 손이 더 가는 건 사실이었다(앱 생성 → 권한 설정 → 조직에 설치 → private key 발급/시크릿 등록, 총 여러 단계) — 그리고 실제로 설정하다가 처음엔 **개인 계정(@iChaeYeong) 소유로 잘못 생성**됐다(조직 설정 화면의 "My GitHub Apps" 링크가 실제로는 개인 계정의 앱 목록 페이지로 연결되는 GitHub UI 함정 때문). 개인 소유 앱은 "특정 개인 계정에 종속되지 않기" 취지에 안 맞아서 삭제하고, `https://github.com/organizations/qKet/settings/apps/new` URL로 직접 들어가서 조직 소유로 재생성했다.

## 구현

- App: `qket-ci-bot`, Owned by `qKet`, App ID `4543569`, `CD` 레포에만 설치
- 시크릿: `QKET_CI_APP_ID`, `QKET_CI_APP_PRIVATE_KEY` — `qKet/backend`, `qKet/frontend` 양쪽에 등록 (`gh secret set <이름> --repo <레포> < file.pem` 또는 GitHub 웹의 Settings → Secrets and variables → Actions에서 직접)
- CI 워크플로우 쪽 흐름: 이미지 빌드/ECR push → `actions/create-github-app-token@v1`로 토큰 발급 → `actions/checkout@v4`로 `qKet/CD`를 그 토큰으로 checkout → `sed`로 `helm/values.yaml`의 해당 image 줄만 정확히 교체(idempotent) → `git commit`/`push`

## 관련
- [[../decisions/2026-08-06-ci-tool-github-actions]]
