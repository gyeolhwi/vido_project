---
description: 변경사항을 기능별로 그룹화하여 자동 커밋
allowed-tools: Bash(git add:*), Bash(git status:*), Bash(git diff:*), Bash(git commit:*), Bash(git reset:*), Bash(git log:*), Bash(git branch:*), Bash(date:*)
disable-model-invocation: true
---

# Commit Auto - 기능별 그룹 자동 커밋

## Context

- 현재 날짜: !`date +%Y-%m-%d`
- 현재 브랜치: !`git branch --show-current`
- 변경 파일 목록: !`git diff --staged --name-status`
- 최근 커밋: !`git log --oneline -5`

<staged_diff>
!`git diff --staged`
</staged_diff>

## Task

staged diff가 비어 있으면 "스테이징된 변경사항이 없습니다"를 알리고 중단한다.

### 1. 기능별 그룹 식별

diff 내용을 읽고 **변경 목적이 같은 파일끼리** 그룹으로 묶는다.
파일 경로나 확장자가 아닌, 변경의 의도를 기준으로 판단한다.

### 2. 그룹별 커밋 메시지 생성

각 그룹에 대해 아래 형식으로 메시지를 생성한다:

```
<type>(<scope>): <subject> - YYYY-MM-DD

<body>
```

### 3. 자동 커밋 실행

- 그룹이 1개면: 바로 `git commit -m "<message>"`
- 그룹이 2개 이상이면: `git reset HEAD` → 그룹 단위로 `git add` → `git commit` 순차 실행
- 모든 커밋 완료 후 `git log --oneline -N`으로 결과를 보여준다

## Commit Convention

**형식:** `<type>(<scope>): <subject> - YYYY-MM-DD`

**Type:** feat | fix | refactor | style | docs | test | chore | perf | ci

**Subject:** 한국어, 50자 이내, 마침표 없음, 명령형

**Body:** 변경 이유와 핵심 내용을 간결하게

## PowerShell 한글 인코딩

PowerShell에서 한글이 깨지면 `-m` 대신 `git commit`으로 에디터를 열거나 `git commit -F msg.txt`(UTF-8)를 사용한다.
