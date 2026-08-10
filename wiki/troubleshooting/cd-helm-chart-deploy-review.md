---
title: CD Helm 차트를 "실제로 배포하면 돌아가는지" 리뷰하며 찾은 문제들
category: troubleshooting
tags: [cd, helm, argocd, docker, backend, frontend]
created: 2026-08-10
updated: 2026-08-10
---

# CD Helm 차트를 "실제로 배포하면 돌아가는지" 리뷰하며 찾은 문제들

`qKet/CD` write-back 파이프라인(GitHub App 기반, [[decisions/2026-08-10-cd-writeback-github-app]])이 이미지 태그는 잘 갱신하고 있어서 "여기까진 끝났다"고 생각했는데, ArgoCD가 그 값으로 실제 배포를 하면 정상 동작할지 코드 자체를 리뷰해보니 별개의 문제 4가지가 나왔다. 하나하나는 각자 다른 원인이지만, 전부 "CI/CD가 도는 것"과 "배포 결과물이 실제로 동작하는 것"을 구분하지 않고 넘어가서 생긴 문제라는 공통점이 있어서 한 페이지에 묶는다.

## 문제 1: frontend Deployment에 백엔드 주소 env가 빠져 있었음

### 증상
`frontend/next.config.js`의 `rewrites()`가 `/api/*` 요청을 백엔드로 프록시하는데, 이 프록시 대상 주소를 실제 배포 환경에서 설정하는 부분이 없었다. 배포하면 프론트에서 나가는 모든 API 호출이 100% 실패할 상황.

### 원인
`CD/helm/templates/frontend-deployment.yaml`의 `env` 블록이 통째로 주석 처리돼 있었고, 심지어 주석 안 변수명도 `next.config.js`가 실제로 읽는 `CLUSTER_IP`가 아니라 `BACKEND_URL`로 잘못 적혀 있었다(`next.config.js`는 `CLUSTER_IP` 없으면 `http://localhost:8080`로 폴백 — 컨테이너 자기 자신을 가리키므로 당연히 실패).

### 해결
```yaml
env:
  - name: CLUSTER_IP
    value: "http://qket-backend-service"
```
로 주석 해제 + 이름 수정. `path`는 안 붙임 — `next.config.js`가 destination 만들 때 이미 `/api/:path*`를 뒤에 붙이므로 여기서 `/api`까지 넣으면 `/api/api/...`로 겹친다.

## 문제 2: ArgoCD Application이 CD 레포의 존재하지 않는 경로를 가리키고 있었음

### 증상
`Infra/argocd/qket-cd-app.yaml`이 실제로 적용되면 ArgoCD sync가 100% 실패할 상태였음.

### 원인
CD 레포가 raw manifest(`release/`) 구조에서 Helm 차트(`helm/`) 구조로 바뀌면서 옛 raw manifest는 `release(backup)/`으로 이름이 바뀌었는데, `qket-cd-app.yaml`의 `spec.source.path`는 여전히 `release`를 가리키고 있었다.

### 해결
`path: helm`로 수정. release 환경은 `helm/values.yaml` 자체가 이미 release 기준 값(`namespace: qket-release` 등)을 직접 담고 있어서 `valueFiles`를 따로 안 줘도 됨(ArgoCD의 Helm source가 기본으로 `values.yaml`을 씀).

> ⚠️ 미해결: prod용 ArgoCD Application은 아직 안 만들어져 있음. prod는 `values.yaml` + `values-prod.yaml`을 함께 레이어링(`valueFiles: [values.yaml, values-prod.yaml]`)해야 하는데, 이 부분은 이번 리뷰에서 손 안 댐 — prod 배포를 실제로 시작할 때 다시 확인 필요.

## 문제 3: frontend Dockerfile이 CI가 이미 끝낸 빌드를 중복으로 다시 하고 있었음

### 증상
CI(`frontend/.github/workflows/CI-release.yml`) 주석엔 "Next.js 빌드를 Docker 안이 아니라 여기서도 한 번 먼저 함 — backend와 동일 패턴"이라고 적혀 있는데, 실제 `frontend/Dockerfile`은 여전히 `deps`/`builder`/`runner` 3단계 멀티스테이지로 `npm install`+`npm run build`를 통째로 다시 돌리고 있었다. CI 빌드 결과물은 그냥 버려지고, 빌드가 두 번 도는 것뿐이라 기능적으로 깨지진 않지만 의도와 실제가 어긋난 상태였음(빌드 시간 낭비, "무엇이 실제로 배포되는 코드인지" 혼선 소지).

### 원인
CI에 write-back 스텝을 추가할 때 CI 쪽 빌드 스텝은 새로 추가했지만, Dockerfile을 backend와 같은 single-stage로 정리하는 건 같이 안 됐던 것으로 보임.

### 해결
backend Dockerfile과 동일한 패턴으로 single-stage로 재작성 — CI가 만든 `.next/standalone`(+`.next/static`, `public`)을 그대로 담기만 함:
```dockerfile
FROM node:20-alpine
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
ENV CLUSTER_IP=http://qket-backend-service
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs
COPY public ./public
COPY --chown=nextjs:nodejs .next/standalone ./
COPY --chown=nextjs:nodejs .next/static ./.next/static
USER nextjs
EXPOSE 3000
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"
CMD ["node", "server.js"]
```
Dockerfile 안 `ENV CLUSTER_IP`는 K8s Deployment의 env가 덮어쓰므로, 그 설정을 깜빡했을 때의 안전망 정도의 의미만 있음(문제 1이 실제로 배포되기 전까진 이 안전망도 값이 없었다는 뜻).

### 확인차 backend도 같이 점검함
backend는 `Dockerfile`/`CI-release.yml` 둘 다 처음부터 제대로 돼 있었음(Dockerfile이 이미 single-stage로 CI가 만든 jar만 담고, CI도 중복 빌드 없음) — frontend만 드리프트가 있었던 것. 단, `EXPOSE 8090`이 실제 `application.yml`의 `server.port: 8080`(K8s Service의 `targetPort`와도 일치)과 안 맞아서 `EXPOSE 8080`으로 정리함 — `EXPOSE`는 문서화 목적일 뿐 실제 라우팅엔 영향 없어서 기능적 버그는 아니었음.

## 문제 4: backend가 실제로 필요로 하는 env var 중 일부가 CD 차트에 아예 안 연결돼 있었음

### 증상
backend Dockerfile/CI가 정상이라는 걸 확인한 김에, `application.yml`이 `${...}`로 참조하는 환경변수 전체를 CD 차트(`configMaps`/`secrets`)가 실제로 다 채워주는지 대조해봄.

### 원인
| 그룹 | 값 출처 | 상태 |
|---|---|---|
| `DB_HOST`/`USERNAME`/`PASSWORD` | ESO → `db-secrets` | ✅ |
| `DB_PORT`/`NAME`, `REDIS_PORT`, `AWS_REGION` | Terraform → `app-config` | ✅ |
| `REDIS_HOST` | ESO → `redis-secrets` | ✅ |
| `S3_BUCKET`/`CLOUDFRONT_DOMAIN` | Terraform → `storage-config` | ✅ |
| `APP_BASE_URL` | 없었음 | ❌ → 이번에 수정 |
| `GOOGLE_CLIENT_ID`/`SECRET`, `KAKAO_*`, `NAVER_*` | 없음 | ❌ 미해결 |
| `TOSS_SECRET_KEY` | 없음 | ❌ 미해결 |

`APP_BASE_URL`은 `OAuthController`가 소셜 로그인 완료 후 프론트로 리다이렉트할 때 쓰는 값(`application.yml` 주석: "소셜 로그인 완료 후 리다이렉트할 프론트엔드 origin")인데, 기본값이 `http://localhost:3000`이라 실배포에선 조용히 틀린 주소로 리다이렉트됐을 것.

OAuth 3사 client-id/secret과 `TOSS_SECRET_KEY`는 기본값이 각각 빈 문자열 / 소스에 박힌 테스트용 키(`test_gsk_docs_...`)라서, 이 상태로 배포하면 **소셜 로그인은 완전히 동작 안 하고, 결제는 실제 토스 계약 키가 아니라 테스트 키로 조용히 처리**된다.

### 해결
`APP_BASE_URL`은 이미 `values.yaml`/`values-prod.yaml`에 있는 `ingress.host`를 그대로 재사용해서 해결(환경별로 값을 중복 관리할 필요 없게):
```yaml
# CD/helm/templates/backend-deployment.yaml
env:
  - name: APP_BASE_URL
    value: "https://{{ .Values.ingress.host }}"
```
`helm template`로 렌더링 결과가 release 기준 `https://dev.jun979.click`로 정확히 나오는 것까지 확인함.

> ⚠️ 미해결: OAuth 3사 client-id/secret 6개 + `TOSS_SECRET_KEY`는 팀이 각 서비스(Google/Kakao/Naver 개발자 콘솔, Toss 대시보드)에서 실제로 발급받은 값이 있어야 채울 수 있어서, 이번엔 손 안 댐. 값이 준비되면 db-secrets/redis-secrets와 같은 패턴(ESO가 Secrets Manager → K8s Secret으로 동기화, [[eks-destroy-layer-separation]] 참고)으로 새 Secret(예: `oauth-secrets`)을 만들고 `backend.secrets`에 추가하는 방향이 기존 아키텍처와 제일 맞는다 — 다만 이 값들은 RDS 비밀번호처럼 Terraform이 자동 생성하는 값이 아니라 사람이 발급받아 넣는 값이라, Secrets Manager 쪽 시드는 `TF_VAR_*` 환경변수나 gitignore된 tfvars로 채우는 방식이 필요함(평문으로 `.tf`에 커밋하면 안 됨).

## 재발 방지

- **"CI가 통과한다" ≠ "배포하면 동작한다"**: 이번에 나온 4가지 문제 전부 CI 로그만 봐서는 안 드러나는 것들이었음(빌드는 성공, 이미지도 push됨, write-back도 됨 — 근데 그 이미지를 실제로 띄우면 런타임에 깨짐). CD 차트나 배포 매니페스트를 바꿀 때는 "이 값이 실제 코드가 읽는 이름/형식과 일치하는가"를 코드(`application.yml`, `next.config.js` 등) 기준으로 직접 대조하는 습관이 필요함.
- **env var 목록은 소스 오브 트루스(`application.yml`의 `${...}`)에서 역으로 대조**: 새 외부 연동(OAuth provider 추가, 결제 PG 추가 등)을 넣을 때마다 `grep -n '\${' application.yml`로 전체 목록을 뽑아서 CD 차트의 `configMaps`/`secrets`와 대조하는 걸 체크리스트로 삼는 게 좋음.
- Dockerfile과 CI의 "빌드를 어디서 하는가"에 대한 서술은 반드시 같이 맞춰야 함 — 한쪽만 바뀌면(이번엔 CI만 바뀌고 Dockerfile은 안 바뀜) 겉보기엔 멀쩡히 배포되는 것처럼 보이면서 실제로는 낭비/혼선이 생김.

## 관련
- [[decisions/2026-08-10-cd-writeback-github-app]]
- [[eks-destroy-layer-separation]]
- [[runbook/daily-infrastructure-toggle]]
