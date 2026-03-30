---
title: "[ Context7 + GitHub MCP 콤보 ]"
date: 2026-03-27
tags: [AI에이전트, Context7, MCP, claudecode, github]
original_url: https://velog.io/@xorms/Context7-GitHub-MCP-콤보
---

# Context7 + GitHub MCP 콤보 — 문서 참조부터 PR까지 프롬프트 한 줄로

## 들어가며

지난 편에서 GitHub MCP를 연결해 자연어만으로 레포 생성, 이슈 관리, PR 생성, 머지까지 처리하는 환경을 구축했습니다.
이번 편에서는 Context7 MCP와 GitHub MCP를 **동시에** 활용하는 콤보 실습을 진행합니다.

콤보로 달라지는 것은 명확합니다. 기존에는 "문서 찾기 → 코드 작성 → git 명령어 → PR 생성"이 4단계였다면, 콤보 이후엔 **프롬프트 한 줄로 전부 처리**됩니다.

---

## MCP 콤보란 무엇인가

두 MCP의 역할을 먼저 정리합니다.

- **Context7**: 라이브러리 공식 문서를 실시간으로 쿼리해 할루시네이션 없이 최신 API 기반 코드를 생성합니다.
- **GitHub MCP**: 브랜치 생성, 파일 커밋, PR 오픈 등 GitHub 서버 작업을 API로 직접 처리합니다.

**기존 워크플로우:**
```
공식 문서 직접 검색 → 코드 작성 → git add/commit/push → GitHub에서 PR 생성
```

**콤보 이후:**
```
프롬프트 한 줄 → Context7 문서 참조 → 코드 작성 → GitHub MCP로 브랜치/커밋/PR 자동
```

---

## MCP scope 개념

실습 전에 반드시 짚고 넘어가야 할 개념이 있습니다. 바로 MCP scope입니다.

`claude mcp add` 명령어로 MCP를 등록할 때 적용 범위를 지정할 수 있습니다.

| scope | 적용 범위 | 명령어 옵션 |
|---|---|---|
| local | 현재 프로젝트만 | 기본값 (옵션 없음) |
| user | 내 계정 전체 (모든 프로젝트) | `--scope user` |
| project | 팀 공유 (.mcp.json) | `--scope project` |

### "있는데 없는 것처럼" 동작하는 함정

이번 실습 중 이전 프로젝트 디렉토리에서 등록한 MCP가 새 프로젝트에서 `claude mcp list`에 뜨지 않는 현상을 겪었습니다.
원인은 간단합니다 → 기본값인 `local` scope로 등록하면 해당 프로젝트 경로에서만 적용되기 때문입니다.

Context7, GitHub MCP처럼 어느 프로젝트에서나 쓰는 범용 MCP는 `--scope user`로 등록하는 것을 추천드립니다.

---

## 실습 1 — user scope로 재등록

```bash
# 기존 등록 제거
claude mcp remove context7
claude mcp remove github

# user scope로 재등록
claude mcp add context7 --scope user \
  -- npx -y @upstash/context7-mcp@latest

claude mcp add github --scope user \
  -e GITHUB_PERSONAL_ACCESS_TOKEN=발급받은토큰 \
  -- npx -y @modelcontextprotocol/server-github
```

등록 후 확인합니다.

```bash
claude mcp list
# context7: ✓ Connected
# github: ✓ Connected
```

---

## 실습 2 — 레포 생성부터 PR까지

### 레포 생성 및 로컬 세팅

```
GitHub MCP를 사용해서 새 레포를 만들어줘.
- name: mcp-combo-practice
- description: Context7 + GitHub MCP 콤보 실습
- private: false
- auto_init: true
```

### 콤보 프롬프트 실행

```
use context7 to look up the latest Express.js documentation,
then create src/index.js with the following:
- GET /health → { status: 'ok', timestamp: 현재시간 }
- GET /users  → 더미 유저 배열 3개 반환
- POST /users → body에서 name 받아서 { message: 'created', user: {...} } 반환

After writing the code, use github mcp to:
1. create a new branch named 'feat/express-api'
2. commit all files (src/index.js, package.json, .gitignore)
3. open a pull request to main with title 'feat: Express.js REST API 기본 엔드포인트 추가'
```

### MCP 호출 로그

```
context7 - resolve-library-id (MCP) ✅
context7 - query-docs (MCP) ✅
github - create_or_update_file (MCP) ✅
github - add_issue_comment (MCP) ✅
```

여기서 눈에 띄는 것이 마지막 줄입니다.
`add_issue_comment`는 프롬프트에 지시하지 않은 단계입니다.
Claude가 PR을 생성한 뒤 "어떤 문서를 참조해서 이 코드를 작성했는지"를 PR 코멘트로 자동 등록한 것입니다.
에이전트(agent)가 목표 달성을 위해 스스로 판단해 추가 행동을 취한 사례입니다.

---

## 트러블슈팅(작성자 환경)

### MCP scope 문제 — claude mcp list에 MCP가 안 뜸

새 프로젝트 디렉토리에서 `claude mcp list`를 실행했을 때 등록한 MCP가 표시되지 않는다면 `local` scope로 등록된 것입니다. `claude mcp remove` 후 `--scope user`로 재등록하면 해결됩니다.

### MCP 미설정 시 Bash로 우회하는 현상

MCP가 없으면 Claude가 `git checkout`, `curl -X POST` 같은 Bash 명령어로 GitHub API를 직접 호출하려 합니다.
`(MCP)` 표시 없이 명령어가 실행된다면 MCP 등록 상태를 먼저 확인하세요.

### auto_init 미적용 현상

GitHub MCP의 `create_repository`에서 `auto_init: true`를 지정했음에도 빈 저장소로 생성되는 경우가 있습니다.
클론 시 "빈 저장소를 복제한 것처럼 보입니다" 경고가 뜬다면 README.md 생성을 별도로 요청해 첫 커밋을 만들면 됩니다.

---

## 마무리 및 다음 편 예고

Context7 + GitHub MCP 콤보를 쓰고 나면 개발 사이클의 구조가 달라집니다. 문서를 찾고, 코드를 짜고, 커밋하고, PR을 여는 과정이 프롬프트 한 줄로 처리됩니다.

다음 편에서는 Notion MCP Server 연동을 다룰 예정입니다.
