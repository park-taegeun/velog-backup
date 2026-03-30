---
title: "[ Context7 MCP ]"
date: 2026-03-27
tags: [AI 에이전트, Claude, Context7, MCP, claudecode]
original_url: https://velog.io/@xorms/Context7-MCP
---

# Claude Code에 Context7 MCP 연결하기 — AI 할루시네이션 없이 최신 문서 참조하는 법

## 들어가며

요즘 AI 코딩 어시스턴트를 쓰다 보면 한 가지 패턴이 반복됩니다.

"이 API 어떻게 써요?"라고 물으면 그럴싸한 코드를 바로 뽑아주는데, 막상 실행해보면 deprecated된 문법이거나 버전이 맞지 않아 에러가 납니다.

AI가 자신 있게 틀리는 현상 — 할루시네이션(hallucination)입니다.

저는 MCP 에이전트를 활용해 1인 개발자로서 효율적인 워크플로우를 구성하는 것을 목표로 AI 도구 생태계를 단계별로 탐구하고 있습니다.

그 첫 번째 실습으로 **Claude Code에 Context7 MCP를 연결해 AI가 실시간으로 공식 문서를 참조하며 코드를 생성하는 환경**을 만들었습니다.

이 글에서는 그 과정을 공유합니다.

---

## MCP란 무엇인가

MCP(Model Context Protocol)는 Claude가 외부 도구를 직접 호출할 수 있도록 Anthropic이 설계한 프로토콜입니다.

기존 AI는 텍스트를 받아 텍스트를 돌려주는 구조였습니다. MCP를 사용하면 Claude가 실제 API, 문서 시스템, 데이터베이스에 직접 접근해서 **행동(action)** 할 수 있게 됩니다.

> **비유하자면:** Claude는 두뇌, MCP는 그 두뇌가 사용하는 손발입니다. 두뇌만 있으면 생각은 할 수 있지만 행동은 못 합니다. MCP가 있어야 실제로 무언가를 할 수 있습니다.

이때 Claude가 외부 시스템을 호출하는 행위를 **도구 호출(tool use)** 이라고 합니다.

도구 호출(tool use)은 단순 텍스트 생성이 아니라 정의된 함수를 실행하는 것으로, 결과가 다시 Claude에게 돌아와 최종 답변에 반영됩니다.

---

## Context7이란 무엇인가

### 할루시네이션 문제

AI는 학습 데이터를 기반으로 답변을 생성합니다. 문제는 라이브러리나 프레임워크가 업데이트될 때 발생합니다.

학습 데이터에 없는 최신 API 변경사항은 AI가 알 수 없고 이를 모르는 상태에서도 자신 있게 답변합니다. 이것이 할루시네이션입니다.

**Context7**은 이 문제를 해결합니다. 라이브러리 공식 문서를 실시간으로 쿼리해서 최신 정보를 Claude에게 직접 제공하는 MCP 서버입니다.

### Context7을 써야 할 때 vs 쓰지 않아도 될 때

| 상황 | Context7 필요 여부 |
| --- | --- |
| 특정 라이브러리 API 사용법 (버전별 차이 큼) | 필요 |
| 최신 기능 (Next.js App Router, React 19 등) | 필요 |
| 처음 쓰는 패키지의 기본 사용법 | 필요 |
| 개념 및 원리 설명 요청 | 불필요 |
| 알고리즘 설계, 로직 구성 | 불필요 |
| 코드 리팩토링, 버그 수정 | 불필요 |

---

## 설치 실습(작성자의 환경에 맞춘 단계)

### 사전 요구사항

Context7 MCP 서버는 Node.js 기반으로 실행됩니다. **v18 이상**이 필요합니다.

```
node --version
# v18.x.x 이상이어야 합니다
```

### 1단계: 현재 MCP 연결 상태 확인

```
claude mcp list
# No MCP servers configured
```

아무것도 연결되지 않은 초기 상태를 확인합니다.

### 2단계: Context7 MCP 서버 추가

```
claude mcp add context7 -- npx -y @upstash/context7-mcp@latest
# Added stdio MCP server context7 to local config
# File modified: /Users/{username}/.claude.json
```

`stdio` 방식은 별도의 외부 서버 없이 로컬 프로세스로 실행되는 MCP 서버 유형입니다. 설정은 `~/.claude.json`에 자동 저장됩니다.

### 3단계: 연결 확인

```
claude mcp list
# context7: npx -y @upstash/context7-mcp@latest — ✓ Connected
```

`✓ Connected`가 뜨면 성공입니다.

### 4단계: Claude Code 실행

```
claude
```

처음 실행 시 신뢰 폴더 확인 프롬프트가 뜨면 Enter로 진행합니다.

---

## 실제 동작 확인

Claude Code에 아래 프롬프트를 입력합니다:

```
React의 useEffect 클린업 함수 사용법 알려줘. use context7
```

프롬프트 끝에 `use context7`을 붙이면 Claude가 Context7 MCP를 활성화해 문서를 조회합니다.

### tool use 2단계 호출 과정

Context7은 내부적으로 두 단계의 도구 호출을 실행합니다.

**1단계 — resolve-library-id**

라이브러리 이름을 공식 문서 ID로 변환합니다.

```
"React" → "/websites/react_dev"
```

**2단계 — query-docs**

변환된 ID를 기반으로 react.dev에서 실제 문서 내용을 쿼리합니다.

```
/websites/react_dev → useEffect cleanup 관련 최신 문서 반환
```

### 결과물

react.dev 기반으로 다음 내용이 출력되었습니다:

- 클린업 함수 기본 구조
- 이벤트 리스너 / 타이머 / 구독 해제 패턴
- 클린업 호출 시점 (언마운트 / 의존성 변경 / StrictMode)
- 비동기 요청 취소를 위한 AbortController 활용법

학습 데이터에 의존하는 답변이 아니라 공식 문서를 실시간으로 참조한 답변이라는 점에서 신뢰도가 달라집니다.

---

## 트러블슈팅(작성자 환경)

### 401 Authentication Error

Claude Code 실행 시 아래와 같은 에러가 발생할 수 있습니다.

```
Error: 401 Authentication Error
```

장기간 미사용으로 세션이 만료된 경우입니다.

Claude Code 내부에서 `/login` 명령어를 입력하면 재인증 후 정상 동작합니다.

```
/login
```

### MCP 도구 호출 허락 팝업

처음 MCP tool use가 실행될 때 매번 허락 여부를 확인하는 팝업이 뜹니다.

**"Yes, and don't ask again"** 을 선택하면 이후 자동으로 실행됩니다.

---

## 마무리 및 다음 편 예고

Context7 MCP를 연결하고 나면 Claude가 라이브러리 문서를 직접 조회하면서 코드를 생성합니다.

학습 데이터의 한계를 실시간 문서 참조로 보완하는 방식이라, 버전 의존성이 높은 프로젝트에서 체감 효과가 큽니다.

다음 편에서는 **GitHub MCP 연결**을 다룰 예정입니다. Personal Access Token(PAT) 발급부터 PR 생성, 이슈 관리 자동화까지 Claude Code와 GitHub를 연결하는 과정을 실습합니다.

---
