---
title: "[ Playwright MCP Server ]"
date: 2026-03-30
tags: [AI에이전트, MCP, claudecode, playwright, 브라우저자동화]
original_url: https://velog.io/@xorms/Playwright-MCP-Server
---

# Claude Code로 Playwright MCP 연결하기 — 브라우저를 자연어로 조작하는 AI 에이전트

## 들어가며

지난 편에서 Notion MCP를 연결해 워크스페이스 탐색, 페이지 생성, 개발 내용 자동 기록까지 자연어로 처리하는 환경을 구축했습니다.
이번 편에서는 **Playwright MCP**를 연결합니다.

지금까지 연결한 MCP들의 역할을 정리하면 이렇습니다.
Context7은 문서를 읽고, GitHub은 코드를 관리하고, Notion은 기록을 담당합니다. Playwright MCP는 여기에 하나를 더합니다. **실제 브라우저를 직접 실행하고 조작합니다.**

---

## Playwright MCP란 무엇인가

Playwright는 Microsoft가 만든 브라우저 자동화 프레임워크입니다.
Chromium, Firefox, WebKit(Safari) 세 엔진을 모두 지원하며, 사람이 브라우저를 조작하는 방식(보고, 클릭하고, 입력하고, 확인하는 흐름)을 코드로 동일하게 재현할 수 있습니다.

Playwright MCP Server는 이 Playwright를 MCP 서버로 감싼 것입니다.
자연어 지시가 MCP 도구 호출로 변환되고, 그 결과로 실제 브라우저가 동작하는 구조입니다.

기존 MCP들은 API를 호출하거나 문서를 읽는 방식이었습니다. Playwright MCP는 다릅니다.
실제 브라우저가 열리고, 페이지가 로드되고, 요소가 클릭됩니다.
JavaScript로 렌더링되는 SPA도 실제 브라우저로 처리하기 때문에 API가 없는 서비스에도 접근할 수 있습니다.

---

## Headless vs Headed 모드

Playwright는 두 가지 실행 모드를 지원합니다.

| 구분 | Headless | Headed |
|---|---|---|
| 브라우저 창 | 없음 (백그라운드 실행) | 있음 (눈에 보임) |
| 속도 | 빠름 | 상대적으로 느림 |
| 리소스 | 적게 사용 | 더 사용 |
| 주요 용도 | CI/CD, 서버 자동화 | 디버깅, 개발 중 확인, 시연 |

최신 `@playwright/mcp@latest`는 **headed가 기본값**입니다.
실습 중 Claude가 "현재 headless로 구성되어 창이 표시되지 않는다"고 안내했지만 실제로는 브라우저 창이 정상적으로 열렸습니다.
Claude의 학습 데이터가 구버전 기준이었던 것으로 파악됩니다. headless로 전환하려면 등록 시 `--headless` 플래그를 추가하면 됩니다.

---

## 핵심 도구 소개

Playwright MCP의 주요 도구는 크게 탐색, 읽기, 인터랙션 계열로 나뉩니다.

**탐색:** `browser_navigate`, `browser_go_back`, `browser_go_forward`
**읽기:** `browser_snapshot`, `browser_screenshot`
**인터랙션:** `browser_click`, `browser_type`, `browser_hover`, `browser_select_option`, `browser_drag`
**제어:** `browser_wait`, `browser_press_key`

이 중 특히 구분이 필요한 것이 `browser_snapshot`과 `browser_screenshot`입니다.

| 구분 | browser_snapshot | browser_screenshot |
|---|---|---|
| 반환 형태 | 텍스트 (접근성 트리) | 이미지 |
| Claude 처리 방식 | 구조 파악 후 요소 특정 | vision으로 이미지 분석 |
| 속도 | 빠름 | 느림 |
| 주 용도 | 클릭/입력할 요소 찾기 | 결과 확인, 시각적 검증 |

일반적인 도구 호출 흐름은 다음과 같습니다.

```
navigate → snapshot(구조 파악) → click/type(인터랙션) → screenshot(결과 확인)
```

---

## 연결 실습(작성자 환경)

### 1단계: Playwright MCP 등록 (Headed 모드 — 기본값)

```bash
claude mcp add playwright --scope user -- npx -y @playwright/mcp@latest
# Added stdio MCP server playwright 확인
```

### 2단계: Headless 모드로 전환하려면

```bash
claude mcp remove playwright --scope user
claude mcp add playwright --scope user -- npx -y @playwright/mcp@latest --headless
```

### 3단계: 등록 확인

```bash
cat ~/.claude.json | python3 -c "
import json, sys
d = json.load(sys.stdin)
print(json.dumps(d.get('mcpServers', {}), indent=2))
"
```

`playwright` 항목이 확인되면 준비 완료입니다.

---

## 실전 시나리오 3가지(작성자 환경)

### 시나리오 1 — 스크린샷 저장

```
playwright로 https://velog.io/@xorms/posts 에 접속해서
스크린샷을 찍고 ~/Desktop/velog.png 로 저장해줘
```

→ `browser_navigate → browser_screenshot` 순서로 도구 호출, 실제 이미지 파일이 저장됩니다.

### 시나리오 2 — 폼 입력 자동화 + 에이전트적 오류 복구

```
playwright로 네이버에서 오늘 서울 날씨 검색해서
현재 기온이랑 날씨 상태 알려줘
```

→ `browser_navigate → browser_type` 순서로 실행 중 `browser_type` 결과가 140,203자로 토큰 한도를 초과했습니다.
Claude Code는 스스로 Bash + Python3을 조합해 파일을 읽고 필요한 데이터만 파싱하는 방식으로 자율 우회했고, 최종적으로 날씨 정보(구름많음, 20.1°C, 습도 28%, 동풍 1.9m/s)를 정상 출력했습니다.

### 시나리오 3 — 정보 추출 + dangerously-skip-permissions

Claude Code는 기본적으로 도구 호출 시마다 사용자 승인을 요청합니다.
브라우저 자동화처럼 연속적인 도구 호출이 많은 작업에서는 `--dangerously-skip-permissions` 플래그로 세션 동안 승인을 자동 통과할 수 있습니다.

```bash
claude --dangerously-skip-permissions
```

```
playwright로 https://velog.io/@xorms/posts 에 접속해서
포스팅 목록을 가져오고, 각 글의 제목과 작성일을 정리해줘
```

→ 승인 없이 자동 실행, JavaScript를 활용해 9개 포스팅 제목과 작성일을 표 형태로 추출했습니다.

---

## 실무 활용 시나리오

**1. E2E 테스트 자동화**
"로그인 → 상품 검색 → 장바구니 → 결제" 플로우를 자연어로 테스트합니다.

**2. 웹 스크래핑 / 모니터링**
JavaScript로 렌더링되는 SPA도 실제 브라우저로 처리하기 때문에 API가 없는 서비스에도 접근 가능합니다.

**3. MCP 콤보 파이프라인**
Playwright(테스트) + GitHub(이슈 생성) + Notion(결과 기록)을 연결하면 "UI 테스트 실패 → 자동 이슈 생성 → Notion 기록"까지 풀 사이클 자동화가 가능합니다.

---

## 마무리 및 다음 편 예고

Playwright MCP를 연결하고 나면 Claude Code가 단순한 코드 생성 도구를 넘어 실제 브라우저를 조작하는 에이전트로 동작합니다.

다음 편에서는 PostgreSQL / Supabase MCP 연동을 다룰 예정입니다.
