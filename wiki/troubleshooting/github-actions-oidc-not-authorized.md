---
title: GitHub Actions OIDC — "Not authorized to perform sts:AssumeRoleWithWebIdentity" (근본 원인 미해결)
category: troubleshooting
tags: [infra, terraform, github-actions, oidc, iam]
created: 2026-08-07
updated: 2026-08-08
---

# GitHub Actions OIDC — "Not authorized to perform sts:AssumeRoleWithWebIdentity"

> ⚠️ **인증 자체는 해결됐지만("동작하는 조합을 찾았다"), 왜 `repository`/`ref`/`job_workflow_ref` 커스텀 claim 조건이 실패했는지 근본 원인은 여전히 모른다.** 처음엔 임시방편(ID 와일드카드)으로 남아있던 보안 트레이드오프는 이후 정확한 ID를 박아넣어서 해소함(아래 "남은 리스크" 참고). 비슷한 문제 겪으면 이 문서의 시행착오 순서를 참고하되, "커스텀 claim이 왜 안 되는지"는 여기서 답을 못 얻는다는 것만 알아둘 것.

## 증상

`backend`(`qKet/backend`) 레포에서 `CI-release.yml`이 `aws-actions/configure-aws-credentials@v4`로 OIDC 인증을 시도할 때, "Assuming role with OIDC"를 9~12번 재시도하다가 결국:

```
Error: Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity
```

## 시행착오 순서

### 1차: 기본 `sub` 형식으로 시작 → 실패, 원인 발견

`modules/github-actions-oidc`를 처음 만들 때 흔히 쓰는 기본 형식으로 조건을 걸었음:
```
token.actions.githubusercontent.com:sub = repo:qKet/backend:ref:refs/heads/release
```
계속 Not authorized. 워크플로우에 `actions/github-script`로 실제 OIDC 토큰을 디코딩해서 찍어보는 디버그 스텝을 추가해서 실제 값을 확인:
```
"sub": "repo:qKet@313320752/backend@1323850932:ref:refs/heads/release"
```
**원인 1 발견(최초 추정, 부정확했음 → 아래 "정정" 참고)**: 처음엔 `qKet` 조직이 GitHub의 **"Customize subject claims"** 옵션을 누가 수동으로 켜둔 걸로 추정했음. 실제로는 그게 아니었다 — 아래 "정정: 이건 조직 설정이 아니라 GitHub의 최근 전사 정책 변경" 참고.

> ### 정정 (2026-08-08 추가 조사): 이건 조직이 "켠" 게 아니라 GitHub의 최근 전사 정책 변경
>
> GitHub가 **2026-04-23**에 ["Immutable subject claims for GitHub Actions OIDC tokens"](https://github.blog/changelog/2026-04-23-immutable-subject-claims-for-github-actions-oidc-tokens/)를 발표함 — 조직/레포 이름이 재사용될 수 있다는 문제(예: `qKet` 조직을 삭제하고 다른 사람이 같은 이름으로 새로 만들면 예전 이름 기반 신뢰 정책이 새 조직에 잘못 매칭됨)를 막기 위해, `sub` claim에 불변 숫자 ID(`owner_id`/`repo_id`)를 끼워넣는 새 형식을 도입한 것.
>
> 핵심 룰: **2026-07-15 이후 새로 생성된 레포는 자동으로 새 형식을 씀** — 기존 레포는 명시적으로 옵트인하지 않는 한 영향 없음. 즉 조직이 뭔가를 "켜야" 발동하는 게 아니라, 생성 시점이 컷오프 이후면 자동 적용됨.
>
> 실제로 확인해보니:
> ```
> qKet/backend  created_at: 2026-08-05T09:09:55Z
> qKet/frontend created_at: 2026-08-05T08:10:38Z
> ```
> 둘 다 컷오프(7/15) **이후**에 막 만들어진 레포라서, 그냥 이 새 규칙에 자동으로 걸린 것뿐이었음. "qKet이 옵션을 켰다"는 최초 추정은 틀렸다 — 아무도 아무것도 켜지 않았고, 최근에 만든 레포라 기본값이 이미 새 형식이었던 것.
>
> **강사님이 언급한 "본인도 OIDC 이슈가 있다"는 것과의 연관 추정**: 강사님 자료가 이 변경(2026-04-23) 이전 것이라면 당연히 옛날 형식(ID 없는 `repo:org/repo:ref:branch`)을 기준으로 만들어졌을 것. 그런데 강사님이 지금 쓰는 시연용 레포가 최근(7/15 이후)에 새로 만든 거라면, 자료가 가르치는 형식과 실제 토큰 형식이 어긋나서 우리가 1차에서 겪은 것과 똑같은 `Not authorized`가 났을 가능성이 있음(**확인된 사실 아니고 추정** — 강사님 레포 생성일을 직접 확인한 건 아님).
>
> ⚠️ 단, 이 정정은 어디까지나 **`sub` 형식 문제**(4차에서 이미 해결됨)에 대한 설명이다. 아직 미해결인 **`repository`/`ref`/`job_workflow_ref` claim 자체가 조건으로 걸리면 전부 막히는 문제**(5차, 결론 섹션의 SCP/RCP 추정)와는 별개 사안 — immutable ID 변경은 `sub`의 형식만 바꿨을 뿐 토큰에 어떤 claim이 들어있는지, AWS가 그 claim들을 조건으로 평가하는지는 건드리지 않는다. 강사님의 문제가 이걸로 설명된다고 해서 우리의 SCP/RCP 의심 건까지 같은 원인일 거라고 단정할 근거는 없음.

### 2차: `sub` 버리고 `repository`+`ref`로 전환 → Terraform 자체가 거부

`sub` 파싱이 번거로워서, 토큰에 같이 들어있는 더 단순한 claim(`repository: "qKet/backend"`, `ref: "refs/heads/release"`)으로 조건을 바꿈. `terraform apply` 시도하니 **AWS가 정책 자체를 거부**:
```
MalformedPolicyDocument: Trust policy with trusted principal ... must evaluate,
using StringEquals, StringLike or StringEqualsIgnoreCase,
token.actions.githubusercontent.com:sub or token.actions.githubusercontent.com:job_workflow_ref
which is not scoped to all.
```
**원인 2 발견**: AWS가 GitHub OIDC provider를 신뢰하는 Role의 트러스트 정책엔 **`sub` 또는 `job_workflow_ref` 조건이 (전체 허용이 아닌 형태로) 반드시 있어야 한다**는 자체 검증 규칙이 있음.

### 3차: `repository`+`ref`+`job_workflow_ref` 조합 → 정책은 통과, 그런데 여전히 Not authorized

`job_workflow_ref`(`qKet/backend/.github/workflows/*@refs/heads/release`, 파일명만 와일드카드)를 추가해서 위 요구사항을 만족시킴 — `terraform apply` 통과. 근데 **실제 워크플로우 실행은 여전히 Not authorized**.

- 트러스트 정책 JSON을 콘솔에서 통째로 복사해서 실제 토큰 값이랑 필드별로 전부 대조함 (`aud`/`repository`/`ref`/`job_workflow_ref` 4개 다 정확히 일치 확인) — 정책 자체엔 문제 없어 보였음.
- CloudTrail에서 `AssumeRoleWithWebIdentity` 이벤트 확인 → `errorCode: AccessDenied`, `errorMessage: "An unknown error occurred"` — AWS가 의도적으로 뭉뚱그리는 메시지라 더 이상 단서 없음.
- SCP(Service Control Policy) 의심했으나, 이 IAM 계정으론 SCP 조회 권한 자체가 없어서 확인 불가(교육용 계정이라 Organizations 관리자가 따로 있을 가능성).

### 4차(성공): `sub` 하나만, ID 부분을 와일드카드로

원인 규명을 포기하고, **CloudTrail에 실제로 정확히 찍혔던 값**(`principalId`)을 근거로 `sub` 조건 하나만 다시 사용 — 단, 불변 ID 부분만 와일드카드로 열어둠:
```
token.actions.githubusercontent.com:sub (StringLike) = repo:qKet@*/backend@*:ref:refs/heads/release
```
→ **바로 성공.** `Authenticated as assumedRoleId ...`, ECR 로그인까지 정상 진행.

### 5차(추가 진단): `job_workflow_ref` 단독으로도 테스트 → 역시 실패

혹시 3차 실패가 `repository`+`ref`+`job_workflow_ref`를 **한꺼번에** 걸어서 생긴 문제였을 수도 있어서(`job_workflow_ref` 하나만으로도 레포+브랜치 정보가 다 들어있어서 이론적으론 그거 하나로 충분함), `aud`+`job_workflow_ref` 딱 2개 조건으로만 다시 걸어서 격리 테스트함:
```
token.actions.githubusercontent.com:job_workflow_ref (StringLike) = qKet/backend/.github/workflows/*@refs/heads/release
```
→ **역시 Not authorized.** 다시 `sub` 조건으로 복구함.

## 결론: 왜 됐는지는 모름, 근데 패턴은 뚜렷함

`repository`/`ref`/`job_workflow_ref` — **단독이든 조합이든, `sub`/`aud` 말고 다른 claim을 조건으로 걸면 뭘 걸어도 전부 실패**했다. 반대로 `sub`/`aud`만 쓰면 매번 성공한다.

- 정책 내용은 매번 실제 토큰 값이랑 정확히 대조했음(오타/형식 문제 아니었음)
- CloudTrail도 일반 `AccessDenied`만 보여줘서 더 팔 방법이 없었음
- **다른 AWS 계정/조직에서는 `repository`/`ref` 같은 커스텀 claim이 널리 쓰이고 잘 동작하는 게 문서화되어 있음** — 그래서 "AWS가 원래 sub/aud만 지원한다"는 일반적인 한계가 아니라, **이 계정에만 뭔가 `sub`/`aud` 외 조건 키 평가를 막는 설정(SCP/RCP 등)이 걸려있을 가능성이 가장 유력**한 가설임 (확정은 아님)

## 다음 단계 — (예정) 관리자에게 문의할 내용

지금 쓰는 `sub` 기반 해법은 **임시방편**이다. 원래 의도(GitHub 공식 권장 방식)는 `repository`/`ref`/`job_workflow_ref` 같은 사람이 읽기 쉬운 claim이었는데 이 계정에서 원인불명으로 막혔다 — 이 IAM 계정으론 SCP/RCP 조회 권한이 없어서 더 못 팠다. AWS Organizations 관리자(교육 계정 운영진)한테 다음을 그대로 들고 가서 물어볼 것:

**1. 실패했던 정책들 (`sub`/`aud` 말고는 뭘 걸어도 실패)**

조합으로도, 단독으로도 다 실패했다 — 아래 두 버전 다 `Not authorized`:
```json
// 버전 A: repository+ref+job_workflow_ref 조합
{
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
      "token.actions.githubusercontent.com:repository": "qKet/backend"
    },
    "StringLike": {
      "token.actions.githubusercontent.com:ref": ["refs/heads/release", "refs/heads/main"],
      "token.actions.githubusercontent.com:job_workflow_ref": [
        "qKet/backend/.github/workflows/*@refs/heads/release",
        "qKet/backend/.github/workflows/*@refs/heads/main"
      ]
    }
  }
}
```
```json
// 버전 B: job_workflow_ref 단독 (격리 테스트, 그래도 실패)
{
  "Condition": {
    "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
    "StringLike": {
      "token.actions.githubusercontent.com:job_workflow_ref": [
        "qKet/backend/.github/workflows/*@refs/heads/release",
        "qKet/backend/.github/workflows/*@refs/heads/main"
      ]
    }
  }
}
```
반면 `sub`+`aud`만 쓰면(아래, 지금 실제로 쓰는 것) 항상 성공한다:
```json
{
  "Condition": {
    "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
    "StringLike": {
      "token.actions.githubusercontent.com:sub": [
        "repo:qKet@313320752/backend@1323850932:ref:refs/heads/release",
        "repo:qKet@313320752/backend@1323850932:ref:refs/heads/main"
      ]
    }
  }
}
```

role: `arn:aws:iam::727646470302:role/team5-qket-gha-backend`

**2. 확인 부탁할 것**
- 이 계정(`727646470302`)에 **SCP나 RCP(Resource Control Policy)**가 걸려있는지, 걸려있다면 `sts:AssumeRoleWithWebIdentity`나 특정 조건 키(`token.actions.githubusercontent.com:repository` 등) 사용을 제한하는 조항이 있는지
- CloudTrail 이벤트 `AssumeRoleWithWebIdentity` (요청 ID 예: `eea6626e-6263-4322-b9c7-3cc7e6ca192b`, 2026-08-07T16:54:29Z)를 관리자 권한으로 더 자세히 볼 수 있는지 (Config, Access Analyzer, 또는 AWS Support 케이스로)
- 혹시 이 계정이 특정 IAM 조건 키 사용을 화이트리스트 방식으로만 허용하는 별도 가드레일이 있는지

**3. 만약 SCP/RCP가 원인이 아니라고 확인되면**, 다시 `repository`+`ref`+`job_workflow_ref` 조합으로 돌아가서 (지금의 `sub`+정확한 ID 방식보다 더 읽기 쉽고 GitHub 공식 권장 방식이라) 재시도해볼 것.

## 남은 리스크 → 해결함 (2026-08-07 추가 조치)

처음엔 `repo:qKet@*/backend@*:ref:refs/heads/release`처럼 **불변 숫자 ID 부분을 와일드카드로 열어뒀었다.** 근데 "Customize subject claims"(불변 ID 포함)의 목적 자체가 "조직/레포 이름이 나중에 재사용되더라도(예: `qKet` 조직을 삭제하고 다른 사람이 같은 이름으로 새로 만들어도) 예전 이름 기반 정책이 새 조직에 잘못 매칭되지 않게" 하는 보안 장치인데, **ID를 와일드카드로 열면 이 방어를 스스로 무력화**하는 셈이었다.

그래서 와일드카드 대신 **실제 불변 ID를 정확히 박아넣는 걸로 교체**했다:

```
repo:qKet@313320752/backend@1323850932:ref:refs/heads/release
```

ID는 GitHub 공개 API로 인증 없이 바로 확인 가능(public 레포 기준):
```bash
curl -s https://api.github.com/repos/qKet/backend | jq '.id, .owner.id'
```

`modules/github-actions-oidc`에 `github_owner_id`(조직 공통, 1개)와 레포별 `repository_id` 변수를 추가해서, `platform/main.tf`의 `repos` 맵에서 레포마다 정확한 ID를 넘기도록 바꿈. 이제 `qKet`이라는 이름을 재사용한 다른 조직/레포는 절대 매칭 안 됨 — 와일드카드 쓰기 전 원래 의도한 보안 수준을 그대로 유지하면서 `sub` 인증도 정상 동작.

## 재발 시 참고

- `modules/github-actions-oidc`가 `for_each`로 되어있어서 `frontend` role도 같은 패턴이 자동 적용됨 — 별도 조치 안 해도 됨. 그래도 안 되면 이 문서 순서대로 다시 확인.
- CloudTrail에서 실제 거부 사유(비록 뭉뚱그려져 있어도) 확인하는 법: 이벤트 기록 → 이벤트 이름 `AssumeRoleWithWebIdentity`로 필터.

### OIDC 토큰 claim 직접 까보는 법

GitHub Actions가 발급하는 OIDC ID 토큰은 **JWT**(`header.payload.signature` 3부분 구조)이고, 그 안에 `sub`/`aud`/`repository`/`ref` 같은 필드들을 **claim**이라고 부른다. 아래 스텝을 워크플로우에 임시로 추가하면 그 JWT의 **payload(claim 전체)**를 로그로 바로 볼 수 있다:

```yaml
- uses: actions/github-script@v7
  with:
    script: |
      const token = await core.getIDToken('sts.amazonaws.com');
      const payload = JSON.parse(Buffer.from(token.split('.')[1], 'base64').toString());
      console.log(JSON.stringify(payload, null, 2));
```

`qKet/backend`의 release 브랜치 push에서 실제로 찍혔던 payload 예시 (필드 이름/의미 참고용):

```json
{
  "actor": "iChaeYeong",
  "actor_id": "147784687",
  "aud": "sts.amazonaws.com",
  "event_name": "push",
  "iss": "https://token.actions.githubusercontent.com",
  "job_workflow_ref": "qKet/backend/.github/workflows/CI-release.yml@refs/heads/release",     //이거
  "ref": "refs/heads/release",   //이거
  "ref_type": "branch",
  "repository": "qKet/backend",    //이거 
  "repository_id": "1323850932",
  "repository_owner": "qKet",
  "repository_owner_id": "313320752",
  "repository_visibility": "public",
  "run_id": "31198352329",
  "runner_environment": "github-hosted",
  "sha": "9b01b2e550cb50bef3d69e41e3cec350d31bc9b7",
  "sub": "repo:qKet@313320752/backend@1323850932:ref:refs/heads/release",
  "workflow": "Build & Push (backend, release)",
  "workflow_ref": "qKet/backend/.github/workflows/CI-release.yml@refs/heads/release"
}
```

`repository_owner_id`/`repository_id`가 바로 `sub`에 끼어드는 그 불변 ID다 — "남은 리스크"에서 언급한 "와일드카드 대신 정확한 ID로 좁히기"를 나중에 할 때 이 필드를 그대로 가져다 쓰면 된다.

## 관련
- [[../decisions/2026-08-06-ci-tool-github-actions]]
