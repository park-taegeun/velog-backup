---
title: "[ Claude Code Nested Agent ]"
date: 2026-04-01
tags: [AI에이전트, AI자동화, NestedAgent, TaskTool, claudecode]
velog_url: https://velog.io/@xorms/Claude-Code-Nested-Agent
series: AI 생태계 탐험 로그
---

# Claude Code Nested Agent — AI에게 팀을 만들어주는 법

지난 편에서는 Slack MCP를 연결해 자연어로 채널을 조회하고 GitHub 이벤트를 자동으로 알림 전송하는 봇을 구축했습니다.

이번 편에서는 이 시리즈에서 가장 심화된 구조를 다룹니다.
Nested Agent를 활용하면 Claude 하나가 혼자 일하는 것이 아니라, 여러 서브에이전트를 동시에 띄워 작업을 병렬로 처리하는 팀 구조를 만들 수 있습니다.

## Nested Agent란 무엇인가

지금까지의 실습은 모두 단일 에이전트 구조였습니다. Claude Code 하나가 MCP 도구를 직접 호출하고 순서대로 작업을 처리하는 방식입니다.

Nested Agent(중첩 에이전트)는 구조가 다릅니다.
메인 에이전트가 작업을 분석하고 서브에이전트를 생성해 각각에게 작업을 위임합니다. 팀장이 팀원에게 일을 나눠주는 방식으로 이해할 수 있습니다.

단일 에이전트:
Claude → 작업 A → 작업 B → 작업 C (순차)

Nested Agent:
메인 Claude ─┬→ 서브에이전트 A: 작업 A ┐
             ├→ 서브에이전트 B: 작업 B ┤→ 결과 통합
             └→ 서브에이전트 C: 작업 C ┘ (병렬)

## Task Tool이란

메인 에이전트가 서브에이전트를 생성할 때 사용하는 내부 도구입니다.
각 서브에이전트는 독립된 컨텍스트 윈도우(약 200K 토큰)를 가지며, 메인 세션과 완전히 분리되어 작업합니다. 작업이 완료되면 전체 내용이 아닌 요약(summary)만 메인 에이전트로 반환됩니다. 최대 10개까지 동시 실행이 가능합니다.

## 병렬 vs 순차 처리 판단 기준

모든 작업을 병렬로 처리할 수 있는 것은 아닙니다. 판단 기준은 명확합니다.

| 구분 | 조건 |
|------|------|
| 병렬 처리 가능 | 작업 3개 이상 + 파일 간 겹침 없음 + 공유 상태 없음 |
| 순차 처리 필요 | 작업 간 의존성 있음 OR 같은 파일 수정 OR DB→API→프론트처럼 선후관계 존재 |

서브에이전트끼리는 컨텍스트를 공유하지 않습니다. A가 만든 결과물을 B가 참조해야 하는 상황이라면 병렬 처리는 적합하지 않습니다.

## Step 1 — 기본 병렬 작업 체험

프로젝트 폴더를 생성하고 Claude Code를 실행합니다.

```bash
mkdir ~/nested-agent-practice
cd ~/nested-agent-practice
claude
```

프롬프트를 작성할 때 두 가지를 명확히 해야 합니다.
서브에이전트 수와 각 에이전트의 작업 범위(파일 단위)입니다.

```
use 3 parallel sub-agents to create a simple portfolio site.
- Sub-agent A: Create index.html with a basic portfolio page structure
- Sub-agent B: Create style.css with clean modern styling
- Sub-agent C: Create README.md describing the project structure
Each agent works on its own file only. Do not share state between agents.
```

실행하면 Running 3 agents… 로그와 함께 세 서브에이전트가 동시에 시작됩니다. 각 서브에이전트가 독립된 7.8K 토큰 컨텍스트를 보유하고 있다는 것이 로그에서 확인됩니다.
총 소요 시간은 42초로, 순차 처리 대비 약 3배 빠른 속도입니다.

### 독립 컨텍스트의 한계

한 가지 주목할 결과가 나왔습니다. HTML은 90줄, CSS는 810줄로 생성됐지만 서로 연결되지 않았습니다. CSS 클래스가 HTML에 반영되지 않은 미스매치가 발생했습니다.

원인은 명확합니다. 서브에이전트 B는 서브에이전트 A가 만든 HTML 구조를 알지 못합니다. 독립 컨텍스트는 병렬 처리의 핵심 특성이자 동시에 한계입니다. 의존성이 있는 작업은 순차 처리하거나, 메인 에이전트가 통합 단계를 별도로 실행해야 합니다.

## Step 2 — 커스텀 에이전트 정의

매번 프롬프트에 에이전트 역할을 설명하는 대신, 역할을 파일로 사전에 정의해두는 기능이 커스텀 에이전트(Custom Subagent)입니다. `.claude/agents/` 폴더에 마크다운 파일을 두면 메인 에이전트가 description을 읽고 자동으로 위임 대상을 판단합니다.

```bash
mkdir -p .claude/agents
```

에이전트 파일의 기본 구조는 다음과 같습니다.

```bash
cat > .claude/agents/data-analyst.md << 'EOF'
---
name: data-analyst
description: 데이터 분석 및 인사이트 도출 전문가. 데이터 조회, 통계 분석, 패턴 파악 작업 시 자동 호출.
tools: Read, Write, Bash
model: sonnet
---
당신은 데이터 분석 전문가입니다.
EOF
```

data-analyst, report-writer, api-builder 3개 에이전트 파일을 동일한 방식으로 생성했습니다.

### 트러블슈팅 — 수동 파일 생성 버그

파일을 생성했음에도 `/agents` 명령어에서 "No agents found"가 출력됐습니다. 형식이 정확해도 수동으로 만든 파일을 Claude Code가 인식하지 못하는 알려진 버그입니다. GitHub Issue #4728, #11205, #20931 등 다수의 이슈가 공식으로 등록된 상태입니다.

해결 방법은 TUI를 통한 직접 생성입니다.

```
/agents → Create new agent → Project (.claude/agents/) 선택
```

TUI로 생성한 에이전트는 정상적으로 인식됩니다. 공식 문서에는 파일 직접 생성이 가능하다고 명시되어 있지만 실제 동작과 다른 케이스입니다. 문서보다 실제 동작을 먼저 확인해야 한다는 교훈을 얻었습니다.

## 비용 최적화 전략

서브에이전트가 많아질수록 토큰 비용도 늘어납니다. 환경변수 하나로 이를 최적화할 수 있습니다.

```bash
export CLAUDE_CODE_SUBAGENT_MODEL=claude-sonnet-4-5
```

메인 에이전트는 Opus로 복잡한 판단과 위임을 처리하고, 서브에이전트는 Sonnet으로 실제 실행 작업을 처리하는 구조입니다. 판단은 비싸게, 실행은 저렴하게 분리하는 원칙입니다.

## 핵심 학습 포인트 정리

| 항목 | 내용 |
|------|------|
| Task Tool | 메인이 서브에이전트를 생성하는 핵심 메커니즘 |
| 독립 컨텍스트 | 서브에이전트마다 별도 200K 토큰 윈도우 |
| 결과 반환 | 요약(summary)만 메인으로 전달 |
| 병렬 효과 | 파일 겹침 없으면 동시 실행, 약 3배 속도 향상 |
| 커스텀 에이전트 | TUI(/agents)로 생성해야 정상 인식 |

## 마무리

이번 실습으로 Nested Agent의 구조와 Task Tool 동작 방식, 병렬/순차 처리 판단 기준, 커스텀 에이전트 정의까지 전체 흐름을 익혔습니다.

지금까지 Context7, GitHub, Notion, Playwright, Supabase, Sequential Thinking, Firecrawl, Memory, Slack — 9종의 MCP를 연결하고 다양한 콤보를 실험했습니다. Nested Agent는 이 모든 MCP를 여러 에이전트에 분산해 동시에 처리할 수 있는 구조적 기반이 됩니다.

다음 단계에서는 이 구조를 실전에 적용합니다. Snowflake AI & Data 해커톤 2026 출품작으로 리치고 부동산 데이터와 Snowflake Cortex AI, 그리고 지금까지 학습한 Nested Agent를 조합한 시스템을 설계하고 구현할 예정입니다.
