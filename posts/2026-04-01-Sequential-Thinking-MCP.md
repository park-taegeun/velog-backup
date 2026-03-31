---
title: "[ Sequential Thinking MCP ]"
date: 2026-04-01
tags: [AI에이전트, MCP, SequentialThinking, claudecode, 아키텍처설계]
url: https://velog.io/@xorms/Sequential-Thinking-MCP
---

# Claude Code로 Sequential Thinking MCP 연결하기 — AI에게 단계적으로 생각하게 만드는 법

## 들어가며

지난 편에서 Supabase MCP를 연결해 자연어만으로 PostgreSQL 테이블 설계부터 RLS 정책 적용까지 처리하는 환경을 구축했습니다. 이번 편에서는 조금 다른 결의 MCP를 다룹니다. **Sequential Thinking MCP**입니다.

지금까지 연결한 MCP들은 Claude Code가 "무엇을 할 수 있는지"를 확장했습니다. 이와 달리 Sequential Thinking MCP는 Claude Code가 **"어떻게 생각하는지"**를 바꿉니다.

---

## Sequential Thinking MCP란 무엇인가

Sequential Thinking MCP는 AI가 복잡한 문제를 풀 때 생각의 단계를 명시적으로 나눠서 순서대로 처리하게 해주는 MCP 서버입니다. 일반적인 Claude Code 사용 시 AI는 내부적으로 생각하고 바로 결과를 냅니다. Sequential Thinking MCP는 AI에게 "일단 생각부터 해"를 강제하는 도구입니다.

다른 MCP들과 비교하면 역할의 차이가 명확합니다.

| MCP | 역할 | 비유 |
|-----|------|------|
| Context7 | 최신 문서 참조 | 도서관 |
| GitHub | 코드 저장소 조작 | 서랍장 |
| Notion | 정보 기록/조회 | 노트 |
| Playwright | 브라우저 자동화 | 손발 |
| Supabase | DB 설계 및 조작 | 창고 |
| Sequential Thinking | 사고 구조화 | 두뇌 |

다른 MCP들이 외부 도구와의 연결을 확장한다면 Sequential Thinking은 Claude 자체의 추론 방식을 바꿉니다.

---

## sequentialthinking 도구 구조

이 MCP가 제공하는 도구는 단 하나, `sequentialthinking`입니다. AI가 이 도구를 호출하면 아래 속성으로 사고가 전개됩니다.

| 속성 | 역할 |
|------|------|
| thought | 현재 단계에서 하는 생각 내용 |
| thoughtNumber | 몇 번째 생각인지 |
| totalThoughts | 예상 총 단계 수 (동적으로 변경 가능) |
| nextThoughtNeeded | 다음 단계가 더 필요한지 여부 |
| isRevision | 앞 단계 생각을 수정하는 건지 여부 |

특히 주목할 속성은 `isRevision`입니다. 중간에 앞 단계 생각이 잘못됐다고 판단되면 되돌아가서 수정할 수 있는 구조입니다. 단순히 앞으로만 나아가는 것이 아니라, 추론 중에 오류를 스스로 교정하는 것이 가능합니다. `totalThoughts`도 고정값이 아니라 문제가 복잡해지면 중간에 숫자가 늘어나기도 합니다.

---

## 언제 쓰면 좋은가

Sequential Thinking MCP는 답이 하나로 정해지지 않는 복잡한 문제에서 효과가 드러납니다.

- **아키텍처 설계**: "이 서비스에 어떤 DB를 써야 할까?"
- **버그 원인 분석**: "왜 이 API가 500을 반환하지?"
- **복잡한 리팩토링 계획**: 변경 범위와 순서를 단계별로 정리할 때
- **선택지 비교**: "Redis vs Supabase, 어떤 게 이 상황에 맞나?"

단순한 질문보다 트레이드오프가 있는 의사결정 상황에서 진가가 발휘됩니다.

---

## 설치 및 등록(작성자 환경)

```bash
claude mcp add sequential-thinking --scope user -- npx -y @modelcontextprotocol/server-sequential-thinking
```

등록 확인은 아래 명령어로 합니다.

```bash
cat ~/.claude.json | python3 -c "import json, sys; d = json.load(sys.stdin); print(json.dumps(d.get('mcpServers', {}), indent=2))"
# "sequential-thinking": { "type": "stdio", "command": "npx", ... } 확인
```

이번 실습은 트러블슈팅 없이 설치와 실행 모두 원활하게 완료됐습니다.

---

## 실습

### 프롬프트)

```
Sequential Thinking을 사용해서 아래 질문을 단계적으로 분석해줘.
졸업 프로젝트로 실시간 채팅 기능이 있는 웹앱을 만들려고 해.
백엔드 언어로 Node.js vs Python 중 뭘 선택해야 할까?
각 선택지의 트레이드오프를 분석하고 최종 추천을 내려줘.
```

### 7단계 사고 흐름)

**thought 1** — 문제 정의 + 평가 기준 설정: 실시간 처리 능력, 개발 생산성, 생태계, 성능, 포트폴리오 가치 5가지 기준을 먼저 정의했습니다. 결론을 내기 전에 "무엇을 기준으로 판단할 것인가"를 먼저 세운 것입니다.

**thought 2** — Node.js 단독 분석: 이벤트 루프 기반 비동기 처리, Socket.io 생태계, JavaScript 풀스택 통일성 등을 분석했습니다.

**thought 3** — Python 단독 분석: FastAPI + WebSocket 조합, AI/ML 라이브러리 연동 강점, asyncio 기반 비동기 처리를 분석했습니다.

**thought 4** — 실시간 채팅 관점 심층 비교: 두 선택지를 실시간 처리 요구사항에 특화해 비교했습니다.

**thought 5** — 졸업 프로젝트 특수성 고려 ← 핵심 하이라이트

여기서 예상치 못한 장면이 나왔습니다. Claude가 스스로 이런 질문을 던졌습니다.

> "이 앱이 AI 기능을 원하는가?"

지시하지 않은 질문입니다. 단계를 강제로 나누다 보니 "아직 결론 내기 이르다, 변수가 남았다"는 것을 AI 스스로 인식한 것입니다. 일반적인 응답이었다면 thought 3~4 수준에서 바로 결론이 나왔을 장면입니다.

**thought 6** — 시나리오별 분기 분석: A: 순수 채팅 앱 / B: 채팅 + AI 기능 / C: Python이 익숙한 경우 — 세 시나리오로 나눠 각각의 추천을 도출했습니다.

**thought 7** — 최종 검증 + 추천, `nextThoughtNeeded: false`: 앞선 6단계를 검토한 뒤 최종 추천을 내렸습니다.

---

일반 Claude에게 같은 질문을 던지면 바로 결론부터 냅니다. Sequential Thinking은 "평가 기준 먼저 → 각각 분석 → 비교 → 예외 케이스 → 검증" 순서를 강제로 분리합니다. 사고의 과정이 투명하게 열리는 것이 이 MCP의 핵심입니다.

---

## 마무리 및 다음 편 예고

Sequential Thinking MCP를 연결하고 나면 Claude Code의 응답 방식이 달라집니다. 결론보다 과정이 먼저 나오고, AI가 스스로 변수를 발견하고 질문을 던집니다. 복잡한 기술 의사결정이나 아키텍처 설계처럼 트레이드오프가 있는 문제에서 특히 유용합니다.

다음 편에서는 **Sequential Thinking + GitHub + Notion MCP 콤보 실습**을 다룰 예정입니다. 구조적 사고로 아키텍처를 설계하고, 그 결과를 GitHub 이슈로 등록하고, Notion에 기록하는 풀 파이프라인을 실습합니다.
