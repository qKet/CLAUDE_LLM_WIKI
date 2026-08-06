---
title: decisions 사용법 (ADR)
category: decisions
tags: [meta]
created: 2026-08-06
updated: 2026-08-06
---

# decisions/ — 사용법

이 폴더는 **왜 이렇게 하기로 했는지**를 기록하는 ADR(Architecture Decision Record) 모음이다. `conventions/`나 `architecture/`가 "지금 이렇게 되어있다"를 적는다면, 여기는 "당시 무엇을 고민했고 왜 이걸 골랐는지"를 적는다.

아직 이 프로젝트에서 정식으로 기록된 결정은 없다 — 팀 논의로 뭔가를 확정할 때마다(예: 응답 포맷을 왜 `GlobalResponseAdvice`로 통일했는지, 왜 role 체크를 백엔드에서 이중으로 하기로 했는지 등) 아래 형식으로 페이지를 추가한다.

## 파일명 규칙

`YYYY-MM-DD-짧은-제목.md`

## 템플릿

```markdown
---
title: 결정 제목
category: decisions
status: 확정 | 논의중 | 폐기
date: YYYY-MM-DD
author: 이름
tags: []
---

# 결정 제목

## 배경
왜 이 결정이 필요했는가. 어떤 문제/질문에서 시작됐는가.

## 결정
무엇으로 정했는가. 한 문단으로 명확하게.

## 고려한 대안
- 대안 A — 장단점
- 대안 B — 장단점

## 트레이드오프 / 남은 리스크
이 결정으로 포기한 것, 나중에 재검토가 필요할 수 있는 조건.

## 관련
- [[관련 conventions/architecture 페이지]]
```

## 왜 별도로 관리하나

`conventions/`에 규칙만 적으면 "왜 이렇게 정했는지"가 시간이 지나면 잊혀지고, 나중에 "이거 그냥 바꾸면 안 되나?"라는 질문에 답하기 어려워진다. 결정 당시의 맥락(대안, 트레이드오프)을 남겨두면 재논의할 때 처음부터 다시 고민하지 않아도 된다.
