---
title: Git & GitHub
category: tech
tags: [Git, GitHub, 버전관리, 협업]
created: 2026-05-23
updated: 2026-05-23
sources: [Git_수업강좌정리.txt]
---

# Git & GitHub

## 한 줄 요약
Git은 분산 버전 관리 시스템. 코드 변경 이력을 추적하고 브랜치로 병렬 작업을 지원.

---

## 3-Area 모델 (핵심 개념)

```
작업디렉터리       스테이지(Index)        로컬 저장소
(Working Dir)  →  (Staging Area)   →   (Repository)
               git add              git commit
```

- **작업디렉터리**: 실제 파일을 수정하는 곳
- **스테이지**: 커밋할 파일을 모아두는 임시 공간
- **로컬 저장소**: 커밋(버전)이 저장되는 곳 (`.git` 폴더)

---

## 기본 워크플로우

```bash
git init                    # 저장소 생성 (.git 폴더 생성)
git status                  # 현재 상태 확인

git add 파일명               # 파일을 스테이지에 올림
git add .                   # 현재 디렉터리 전체

git commit -m "메시지"       # 버전 생성
git commit -am "메시지"      # add + commit 한번에 (tracked 파일만)

git log                     # 커밋 이력 확인
git log --oneline           # 한줄 요약
git log -p                  # 변경 내용 포함
```

---

## 되돌리기

### 커밋 전 되돌리기
```bash
git restore 파일명               # 작업디렉터리 변경 취소
git restore --staged 파일명      # git add 취소 (unstage)
```

### 커밋 후 되돌리기

```bash
# 히스토리에서 제거 (로컬에서만 사용 권장)
git reset --soft HEAD^       # commit만 취소, 스테이지는 유지
git reset --mixed HEAD^      # commit + add 취소, 파일은 유지 (기본)
git reset --hard HEAD^       # 작업디렉터리까지 완전히 되돌림 ⚠️

# 히스토리에 새 커밋으로 되돌리기 (협업 시 권장)
git revert 해시값             # 해당 커밋을 취소하는 새 커밋 생성
```

> **reset vs revert**: 원격에 push된 커밋은 `revert` 사용. `reset --hard`는 이미 공유된 이력을 지우기 때문에 위험.

---

## 비교 (diff)

```bash
git diff                    # 작업디렉터리 ↔ 스테이지
git diff --staged           # 스테이지 ↔ 저장소
git diff HEAD               # 작업디렉터리 ↔ 저장소 (위 둘 합산)
git diff 해시1 해시2         # 특정 두 커밋 비교
```

---

## 브랜치

```bash
git branch                  # 브랜치 목록
git branch 브랜치명          # 브랜치 생성
git switch 브랜치명          # 브랜치 전환
git switch -c 브랜치명       # 생성 + 전환
git branch -D 브랜치명       # 브랜치 삭제
git branch -M 새이름        # 브랜치 이름 변경

git log main..hotfix        # main에 없고 hotfix에만 있는 커밋 확인
```

### 병합 (merge) 3가지 상황

**1. Fast-forward** (main에 추가 커밋 없음)
```
main: A - B
hotfix:     - C - D
병합 후: A - B - C - D (단순 포인터 이동)
```

**2. 병합 커밋** (양쪽에 모두 새 커밋 있음)
```bash
git switch main
git merge hotfix    # 새 병합 커밋 자동 생성
```

**3. 충돌 (Conflict)** (같은 파일의 같은 부분을 양쪽에서 수정)
```
<<<<<<< HEAD
main에서 변경한 내용
=======
hotfix에서 변경한 내용
>>>>>>> hotfix
```
→ 직접 수정 후 `git add` + `git commit`

---

## GitHub (원격 저장소)

```bash
# 원격 저장소 연결
git remote add origin https://github.com/유저/레포.git
git remote -v           # 연격 저장소 확인
git remote rm origin    # 연결 해제

# 업로드
git push -u origin main     # 최초 push (-u: upstream 설정)
git push origin main         # 이후 push

# 다운로드
git pull origin main        # fetch + merge 한번에
git fetch origin main       # 다운로드만 (merge 안 함)
git merge origin/main       # fetch 후 수동 병합
```

### 원격 브랜치
```bash
git branch -r               # 원격 브랜치 목록
git branch -a               # 로컬 + 원격 전체
git push origin --delete 브랜치명    # 원격 브랜치 삭제
git fetch -p origin          # 삭제된 원격 브랜치 동기화
```

### Clone
```bash
git clone URL               # 원격 저장소 전체 복제
git clone URL 디렉터리명    # 지정한 이름으로 복제
```
- clone 시 main 브랜치만 로컬에 생성됨
- 다른 브랜치 사용: `git checkout origin/브랜치명` → `git switch 브랜치명`

---

## .gitignore

버전 관리에서 제외할 파일/디렉터리 지정
```
# 예시
*.log
*.class
/target
.env
node_modules/
```

> 이미 추적 중인 파일은 `.gitignore` 추가해도 자동 제외 안 됨
> → `git rm -r --cached .` 후 다시 add

---

## 커밋 수정 (amend)

```bash
git commit --amend    # 마지막 커밋 메시지 수정 또는 파일 추가
```
⚠️ push된 커밋에 amend 후 force push는 공유 이력을 훼손 — 주의

---

## 관련 개념
- [[GitHub Actions]] — CI/CD 자동화 도구
- [[협업 워크플로우]] — Git Flow, PR 기반 협업 방식
