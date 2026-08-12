---
title: OAuth/Toss 외부 API 키를 Secrets Manager 새 시크릿(`external_api`)으로 관리
category: decisions
status: 확정
date: 2026-08-11
author: 이채영
tags: [secrets-manager, eso, oauth, toss, eks]
---

# OAuth/Toss 외부 API 키를 Secrets Manager 새 시크릿(`external_api`)으로 관리

## 배경

[[../troubleshooting/cd-helm-chart-deploy-review]]에서 미해결로 남겨뒀던 문제 — Google/Kakao/Naver OAuth client-id/secret 6개와 `TOSS_SECRET_KEY`가 CD Helm 차트에 아예 연결돼 있지 않아서, 배포하면 소셜 로그인은 완전히 실패하고 결제는 소스에 박힌 테스트 키로 조용히 처리되는 상태였다. 실제 발급값이 준비되면서 이 값들을 어떻게 클러스터로 넣을지 결정할 시점이 왔다.

## 결정

기존 `db-secrets`/`redis-secrets`와 완전히 같은 패턴 — **Secrets Manager에 새 시크릿을 만들고 ESO(External Secrets Operator)로 K8s Secret에 동기화**한다.

- Secrets Manager 시크릿 이름: `team5-qket-external-api-{release,prod}` (`modules/eso/secrets.tf`, `aws_secretsmanager_secret.external_api`)
- 담는 값(7개, 전부 대문자 키): `GOOGLE_CLIENT_ID`/`GOOGLE_CLIENT_SECRET`, `KAKAO_CLIENT_ID`/`KAKAO_CLIENT_SECRET`, `NAVER_CLIENT_ID`/`NAVER_CLIENT_SECRET`, `TOSS_SECRET_KEY`, `TOSS_CLIENT_KEY`
- 값 자체는 **콘솔에서 사람이 직접 입력** — RDS 마스터 비밀번호처럼 Terraform이 자동 생성하는 값이 아니라 각 서비스(Google/Kakao/Naver 개발자 콘솔, Toss 대시보드)에서 발급받은 값을 그대로 옮겨 적는 것뿐이라, `.tf`에 평문으로 심을 이유도 필요도 없음
- `aws_secretsmanager_secret_version.external_api`에 `lifecycle { ignore_changes = [secret_string] }`을 걸어서, 이후 `terraform apply`를 아무리 반복해도(`04_data`는 매일 아침 재적용 대상) 사람이 콘솔로 채워둔 값이 빈 문자열로 덮어써지지 않게 보호
- ESO `ExternalSecret "external-api-secrets"`가 이 시크릿을 K8s Secret `external-api-secrets`로 동기화 — 단, **`TOSS_CLIENT_KEY`는 이 동기화 대상에서 제외**한다(backend는 안 쓰는 값, 필요한 건 frontend 빌드 시점뿐이라 아래처럼 다른 경로로 전달)

## 고려한 대안

- **기존 `db-secrets`/`redis-secrets`에 그냥 끼워넣기** — 기각. RDS/Redis 연결정보와 외부 API 키는 성격이 다른 시크릿이고(순환 주기, 소유 주체, 유출 시 파급범위가 다름), 하나로 합치면 나중에 값 하나 회전할 때 무관한 값들까지 재배포 트리거가 됨. 총 Secrets Manager 개수가 release+prod 합쳐서 6개→8개로 늘어나는 것도 감수할 정도로 분리 이득이 크다고 판단(사용자 확인: "그럼 prod secretmanager 가 3개 release 가 3개 총 6개인데 괜찮은거야" → "그럼 추가하자").
- **GitHub Secrets에 직접 넣기(프론트 CI용 `TOSS_CLIENT_KEY`)** — 기각. 사용자가 명시적으로 거부("깃 씨크릿은 쓰기싫은데") — 이미 AWS OIDC로 GitHub Actions를 인증하고 있는 구조가 있으니, 굳이 별도 시크릿 저장소(GitHub Secrets)를 추가로 관리 대상에 넣지 않고 Secrets Manager 하나로 일원화하는 방향. 자세한 구현은 [[2026-08-11-frontend-ci-toss-key-secrets-manager]].

## 트레이드오프 / 남은 리스크

- `ignore_changes`로 보호한 값은 **Terraform이 다시는 이 필드를 관리하지 않는다는 뜻** — 값을 바꿔야 할 때(키 로테이션 등) `terraform apply`로는 안 되고 콘솔에서 직접 고치거나 `terraform apply -replace=module.eso.aws_secretsmanager_secret_version.external_api`로 명시적으로 트리거해야 함. 코드에도 이 우회 절차를 주석으로 남겨둠.
- ESO 동기화 주기(기본 1시간)를 타는 값이라, 콘솔에서 값을 바꾼 직후엔 K8s Secret에 반영이 안 될 수 있다 — 실제로 이 지연 때문에 결제 오류를 한 번 겪음, [[../troubleshooting/payment-eso-secret-staleness]] 참고.
- prod workspace는 아직 이 시크릿이 실제로 apply된 적 없음(release만 적용됨) — prod 배포 시작 시 값 재입력 필요.

## 관련
- [[../troubleshooting/cd-helm-chart-deploy-review]]
- [[../troubleshooting/payment-eso-secret-staleness]]
- [[2026-08-11-frontend-ci-toss-key-secrets-manager]]
- [[../architecture/terraform-platform-workload-split]]
