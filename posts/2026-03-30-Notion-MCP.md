---
title: "[ Notion MCP ]"
date: 2026-03-30
tags: [AI에이전트, MCP, claudecode, notion, 업무자동화]
original_url: https://velog.io/@xorms/Notion-MCP
---

# Claude Code로 Notion MCP 연결 — 자연어로 페이지 읽고, 만들고, 기록까지

## 들어가며

지난 편에서 Context7 + GitHub MCP 콤보로 문서 참조부터 PR 생성까지 프롬프트 한 줄로 처리하는 워크플로우를 완성했습니다. 이번 편에서는 여기에 **Notion MCP**를 추가합니다.

Notion MCP가 추가되면 달라지는 것은 하나입니다. 개발 사이클의 마지막 단계인 **문서화**까지 자동화됩니다. 코드를 짜고, PR을 올리고, 그 내용을 Notion에 기록하는 흐름이 하나의 프롬프트로 이어집니다.

---

## Notion MCP란 무엇인가

Notion MCP는 Notion이 공식으로 운영하는 MCP 서버입니다.
Claude Code가 이 서버를 통해 Notion API를 직접 호출할 수 있게 됩니다.

### stdio vs HTTP transport — 이번 MCP의 핵심 차이점

지금까지 연결한 Context7, GitHub MCP는 `npx`로 로컬 프로세스를 실행하는 **stdio 방식**이었습니다. Notion MCP는 Notion이 직접 운영하는 서버에 HTTP로 연결하는 **HTTP transport 방식**으로, 등록 방법과 인증 방식이 다릅니다.

| 구분 | stdio 방식 (Context7, GitHub) | HTTP transport 방식 (Notion) |
|---|---|---|
| 실행 방식 | npx로 로컬 프로세스 실행 | Notion 서버에 직접 HTTP 연결 |
| 인증 방식 | 환경변수로 토큰 전달 | OAuth (브라우저 로그인) |
| 패키지 설치 | npx로 자동 설치 | 별도 설치 불필요 |
| 등록 명령어 | `claude mcp add -e TOKEN -- npx ...` | `claude mcp add --transport http [url]` |

---

## OAuth 인증이란 무엇인가

OAuth(Open Authorization)는 서드파티 애플리케이션이 사용자 비밀번호 없이 특정 권한만 위임받아 서비스에 접근하는 개방형 인증 프로토콜입니다.

GitHub MCP를 연결할 때는 PAT(Personal Access Token)를 직접 발급하고 환경변수로 주입했습니다. Notion MCP는 OAuth 방식을 사용합니다.
브라우저가 열리며 Notion 계정으로 로그인하면 Claude Code에 Notion 접근 권한이 위임됩니다.

---

## 연결 실습(작성자 환경)

### 1단계: Notion MCP 등록

```bash
claude mcp add --transport http --scope user notion https://mcp.notion.com/mcp
# Added HTTP MCP server notion with URL 메시지 확인
```

`--transport http` 옵션으로 HTTP transport 방식임을 명시하고, `--scope user`로 모든 프로젝트에서 사용 가능하게 등록합니다.

### 2단계: Claude Code 실행 및 인증 상태 확인

```bash
claude
```

실행 후 `/mcp`를 입력하면 현재 연결된 MCP 목록과 상태를 확인할 수 있습니다.

```
notion · △ needs authentication
```

`notion`을 선택하고 Enter를 누르면 브라우저가 열리며 OAuth 인증 흐름이 시작됩니다.

```
Authentication successful. Connected to notion.
```

---

## 실습(작성자 환경) — 3가지 핵심 기능

### 1. 페이지 목록 조회

```
내 Notion workspace에 어떤 pages가 있어?
```

→ `notion-search (MCP)` 호출, 페이지 목록이 표 형태로 출력됩니다.

### 2. 새 페이지 생성

```
'MCP 실습 기록'이라는 제목으로 Notion 페이지 하나 만들어줘.
```

→ `notion-create-pages (MCP)` 호출, 생성된 페이지 URL이 반환됩니다.

이 단계에서 주목할 점은 **마크다운 → Notion 블록 자동 변환**입니다.
Claude Code가 마크다운 형식으로 내용을 작성하면 Notion API를 통해 헤딩, 표, 번호 목록, 인라인 코드 등 Notion 네이티브 블록으로 자동 변환되어 저장됩니다.

### 3. 중첩 페이지 읽기 및 요약

```
'정보처리기사' 페이지 안에 '실기' 페이지 안에 '최신 IT 기술과 응용' 페이지 내용 요약해줘.
```

→ `notion-search → notion-search → notion-fetch` 순서로 자동 수행됩니다.

중첩된 페이지 구조를 자연어 한 줄로 지시했을 뿐인데, Claude Code가 스스로 계층을 파고들어 내용을 가져옵니다. 첫 번째 검색에서 결과가 없으면 쿼리를 스스로 수정해 재시도하는 에이전트적 오류 복구도 확인할 수 있었습니다.

---

## 실무 활용 시나리오

**1. 스펙 문서 → 코드 직통 연결**
Notion에 작성된 기획서나 API 스펙 문서를 읽고 바로 코드를 생성합니다.

**2. 자동 개발 일지**
GitHub MCP 콤보와 연결하면 머지된 PR 목록을 자동으로 Notion 개발 일지에 기록할 수 있습니다.

**3. MCP 콤보 — 풀 사이클**
Context7(공식 문서 참조) + GitHub(브랜치/PR) + Notion(기록)을 동시에 활용하면 "문서 참조 → 코드 작성 → PR → 개발 일지 기록"까지 프롬프트 한 줄로 완성되는 풀 사이클이 구성됩니다.

---

## 마무리 및 다음 편 예고

Notion MCP를 연결하고 나면 개발 워크플로우의 마지막 빈 칸이 채워집니다.

다음 편에서는 **Playwright MCP Server 연동**을 다룰 예정입니다.
Claude Code에서 브라우저를 직접 제어해 웹 페이지 탐색, 스크린샷 캡처, 자동화 테스트까지 자연어 한 줄로 처리하는 워크플로우를 실습합니다.
