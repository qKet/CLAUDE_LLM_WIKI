---
title: Toss 결제 400 에러 — ESO 동기화 지연으로 K8s Secret에 옛날 값이 남아있던 문제
category: troubleshooting
tags: [eso, secrets-manager, toss, payment, k8s]
created: 2026-08-11
updated: 2026-08-11
---

# Toss 결제 400 에러 — ESO 동기화 지연으로 K8s Secret에 옛날 값이 남아있던 문제

## 증상

프론트에서 Toss 결제를 테스트하다가 `/api/payments/confirm`이 400으로 실패.

## 원인

처음엔 "`TOSS_CLIENT_KEY`와 `TOSS_SECRET_KEY`가 서로 짝이 안 맞는 값인가"를 의심했으나(client key `test_gck_docs_...` / secret key `test_gsk_docs_...` — 접두사가 서로 다름), Secrets Manager에 저장된 실제 값을 직접 대조해보니 **페어링 자체는 처음부터 정확히 맞아있었다**. "짝을 이룬다"는 게 두 값이 같아야 한다는 뜻이 아니라(애초에 client key/secret key는 항상 다른 값), Toss가 발급할 때 세트로 묶어서 내려준 한 쌍이 맞는지가 중요한 것 — 그 페어링은 문제가 없었다.

실제 원인은 K8s Secret `external-api-secrets`의 `TOSS_SECRET_KEY` 값이 **stale**했다는 것: 그 자리에 client-key 모양(`test_gck_docs_...`)의 값이 들어가 있었는데, Secrets Manager 쪽은 이미 올바른 값(`test_gsk_docs_...`)으로 갱신돼 있었다. [[../decisions/2026-08-11-external-api-secrets-manager]]의 ESO `ExternalSecret`은 기본 1시간 주기로만 재동기화하기 때문에, Secrets Manager 값을 콘솔에서 고친 직후엔 K8s Secret이 옛날 값을 계속 들고 있는 시간대가 생긴다 — 같은 클래스의 문제가 Google/Kakao/Naver 키에서도 그 전에 한 번 있었음(반복되는 패턴).

## 해결

```bash
kubectl delete secret external-api-secrets -n qket-release
kubectl rollout restart deployment qket-backend -n qket-release
```

Secret을 지우면 ESO가 다음 재조정 루프에서 Secrets Manager의 **현재** 값으로 즉시 재생성한다(1시간을 기다릴 필요 없음). backend 파드는 시작 시점에 env로 값을 읽어가므로, Secret이 새로 생긴 뒤에도 `rollout restart`로 파드를 다시 띄워야 새 값이 실제로 적용된다.

## 재발 방지

- **Secrets Manager 값을 콘솔에서 직접 수정한 직후엔 항상 `kubectl delete secret <name>`으로 강제 재동기화**를 습관화한다 — 1시간 재동기화 주기를 그냥 기다리면 그 사이 배포/테스트가 옛날 값으로 계속 실패한다.
- 값이 안 맞는 것처럼 보이는 에러가 나면, "Secrets Manager의 값"과 "K8s Secret에 실제로 들어있는 값"을 **각각 따로** 확인하는 습관이 필요하다 — 둘이 다를 수 있다는 전제 자체를 놓치면 엉뚱한 곳(이번엔 페어링 자체)을 의심하며 시간을 쓰게 된다.
- `kubectl get secret external-api-secrets -n qket-release -o jsonpath='{.data.TOSS_SECRET_KEY}' | base64 -d`로 실제 파드가 보는 값을 직접 까보는 게 제일 빠른 확인 방법.

## 관련
- [[../decisions/2026-08-11-external-api-secrets-manager]]
- [[cd-helm-chart-deploy-review]]
