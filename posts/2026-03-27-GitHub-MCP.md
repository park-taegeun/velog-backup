---
title: "[ GitHub MCP ]"
date: 2026-03-27
tags: [AI에이전트, MCP, claudecode, github, 개발자도구]
original_url: https://velog.io/@xorms/GitHub-MCP
---

# Claude Code로 GitHub MCP 연결하기 — 자연어로 이슈/PR/머지까지

## 들어가며

지난 편에서 Claude Code에 Context7 MCP를 연결해 AI가 공식 문서를 실시간으로 참조하며 코드를 생성하는 환경을 구축했습니다.

이번 편에서는 그 다음 단계로 **GitHub MCP**를 연결합니다.

GitHub MCP를 연결하면 달라지는 것은 단순합니다.

터미널 명령어 없이 자연어만으로 GitHub의 전체 워크플로우를 처리할 수 있게 됩니다.

---

## GitHub MCP란 무엇인가

GitHub MCP는 GitHub에서 공식으로 제공하는 MCP 서버입니다.

Claude Code가 이 서버를 통해 GitHub API를 직접 호출할 수 있게 됩니다. 기존에는 `gh` CLI나 웹 UI를 통해 직접 수행하던 작업들을 Claude에게 자연어로 지시하는 것만으로 처리할 수 있게 됩니다.

### Git Bash vs GitHub MCP — 역할 구분

처음 GitHub MCP를 접하면 "그럼 git 명령어도 대신 해주나요?"라는 질문이 자연스럽게 나옵니다. 결론부터 말하면, 역할이 명확히 구분됩니다.

| 작업 | Git Bash (로컬) | GitHub MCP (API) |
| --- | --- | --- |
| add, commit, push | ✅ Bash로 처리 | ❌ 불가 |
| 이슈 생성/조회 | ❌ 불가 | ✅ MCP로 처리 |
| PR 생성/머지 | ❌ 불가 | ✅ MCP로 처리 |
| 레포 생성 | ❌ 불가 | ✅ MCP로 처리 |
| 브랜치 조회/생성 | 로컬만 | ✅ 원격 브랜치 처리 |

`add`, `commit`, `push`는 로컬 저장소를 다루는 git 작업이라 Bash로 처리됩니다.

이슈, PR, 레포 생성처럼 GitHub 서버와 통신하는 작업은 MCP로 처리됩니다.

실전에서는 이 둘을 콤보로 활용해 "코드 작성 → 커밋 → PR 생성"을 한 번에 지시할 수 있습니다.

---

## 이슈(Issue)란 무엇인가

이슈(Issue)는 GitHub에서 제공하는 할 일, 버그, 토론 관리 도구입니다.

개발 워크플로우의 시작점으로 "이 작업을 왜 했는지"의 히스토리를 남기는 역할을 합니다.

일반적인 워크플로우는 다음과 같습니다.

```
이슈 생성 → 브랜치 생성 → 코드 작성 → PR 생성 → 머지 → 이슈 닫기
```

"혼자 하는 프로젝트인데 이슈까지 써야 하나?"라는 생각이 들 수 있습니다.

하지만 솔로 프로젝트에서도 이슈 기반으로 작업하면 포트폴리오 설명이 풍부해집니다.

커밋 히스토리만 있는 레포보다 이슈와 PR이 남아있는 레포가 "이 사람이 어떻게 생각하며 개발했는지"를 보여주기 때문입니다.

---

## 사전 준비 — PAT 발급(작성자 환경)

GitHub MCP는 GitHub API를 인증하기 위해 PAT(Personal Access Token)가 필요합니다. PAT는 비밀번호 대신 사용하는 인증 수단으로, 권한(scope)을 세밀하게 설정할 수 있다는 장점이 있습니다.

**발급 순서:**

1. https://github.com/settings/tokens 접속
2. **Generate new token (classic)** 선택
3. 아래 권한 체크
   - `repo` (전체)
   - `read:org`
   - `read:user`
   - `gist`
4. 만료일: 90일 설정
5. 발급된 토큰(`ghp_...`) 복사 — **페이지를 벗어나면 재확인 불가**

> ⚠️ 토큰은 채팅창, 스크린샷 등 외부에 절대 노출하지 마세요. 노출된 경우 즉시 무효화 후 재발급해야 합니다.

---

## 설치 실습

### 1단계: GitHub MCP 등록

```
claude mcp add github \
  -e GITHUB_PERSONAL_ACCESS_TOKEN=발급받은토큰 \
  -- npx -y @modelcontextprotocol/server-github
```

> ⚠️ 명령어에 `ghp_`를 직접 타이핑한 뒤 복사한 토큰을 붙여넣으면 `ghp_ghp_XXX` 형태로 중복 입력됩니다. 토큰 앞에 아무것도 입력하지 않고 붙여넣기만으로 입력하세요.

### 2단계: 등록 확인

```
claude mcp get github
# Status: ✓ Connected
```

### 3단계: 프로젝트 디렉토리에서 실행

```
mkdir ~/github-mcp-test
cd ~/github-mcp-test
claude
```

홈 디렉토리(`~`)에서 Claude Code를 실행하면 MCP 도구 대신 Bash 명령어로 우회하는 현상이 발생합니다. 반드시 프로젝트 디렉토리를 만들고 해당 경로에서 실행하세요.

---

## 전체 워크플로우 실습

Claude Code에 아래 순서로 자연어로 지시합니다. 각 단계에서 `(MCP)` 표시가 뜨면 GitHub MCP를 통해 API가 호출된 것입니다.

**레포 생성**

```
GitHub에 'mcp-test'라는 이름으로 public 레포 만들어줘
```

→ `github - create_repository (MCP)` 호출 확인

**이슈 생성**

```
mcp-test 레포에 이슈 만들어줘.
제목: 'GitHub MCP 연결 테스트'
내용: 'Claude Code에서 GitHub MCP를 통해 이슈를 생성하는 테스트입니다.'
```

→ `github - create_issue (MCP)` 호출 확인

**브랜치 생성**

```
mcp-test 레포에 'feature/test-branch' 브랜치 만들어줘
```

→ `github - create_branch (MCP)` 호출 확인

**PR 생성**

```
feature/test-branch를 main으로 머지하는 PR 만들어줘.
제목: '테스트 PR'
```

→ `github - create_pull_request (MCP)` 호출 확인

**머지**

```
mcp-test 레포의 PR #2 머지해줘
```

→ `github - merge_pull_request (MCP)` 호출, `merged: true` 확인

### 에이전트적 문제 해결

이번 실습에서 인상적이었던 부분은 Claude가 에러 상황에서 스스로 원인을 파악하고 다음 단계를 진행한 것입니다.

- **"Git Repository is empty"** 에러 발생 → README.md를 자동으로 커밋한 뒤 브랜치 생성 재시도
- **"No commits between branches"** 에러 발생 → test.md 파일을 자동으로 추가한 뒤 PR 생성 재시도

단순한 명령 실행이 아니라 에러를 해석하고 해결책을 찾아 스스로 진행하는 방식, 이것이 Agent적 문제 해결입니다.

---

## 트러블슈팅(작성자 환경)

### 토큰 중복 입력 문제

명령어에 `ghp_`를 직접 타이핑한 뒤 복사한 토큰(`ghp_XXX`)을 붙여넣으면 `ghp_ghp_XXX` 형태로 저장됩니다. 이 경우 인증이 실패합니다.

```
claude mcp remove github
# 이후 토큰 앞에 아무것도 입력하지 않고 붙여넣기만으로 재등록
```

### 홈 디렉토리 실행 시 Bash 우회 현상

홈 디렉토리(`~`)에서 Claude Code를 실행하면 GitHub MCP 대신 `git config`, `gh api` 같은 Bash 명령어로 처리하려는 현상이 발생합니다. `(MCP)` 표시 없이 명령어가 실행된다면 실행 경로를 확인하세요.

---

## 마무리 및 다음 편 예고

GitHub MCP를 연결하고 나면 Claude Code가 GitHub의 실질적인 조작 레이어가 됩니다.

이슈 생성부터 PR 머지까지 전체 워크플로우를 자연어 하나로 처리할 수 있고, 에러 상황에서도 스스로 해결책을 찾아 진행합니다.

다음 편에서는 **Context7 + GitHub MCP 콤보**를 다룰 예정입니다. 최신 공식 문서를 참조하며 코드를 작성하고, 그 결과를 바로 PR로 올리는 워크플로우를 실습해보겠습니다.

---
