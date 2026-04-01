---
title: "[ Firecrawl MCP + 콤보 실습]"
date: 2026-04-01
tags: [AI자동화, FireCrawl, MCP, claudecode, 웹크롤링]
series: AI 생태계 탐험 로그
velog: https://velog.io/@xorms/Firecrawl-MCP-콤보-실습
---

# [ Firecrawl MCP + 콤보 실습]

Firecrawl MCP를 연결해 웹 페이지를 마크다운으로 변환하고, 4 MCP 콤보로 AI 뉴스 모니터링 파이프라인까지 구축하는 과정의 실습 기록

---

## Claude Code로 Firecrawl MCP 연결하기 — 웹 페이지를 AI가 읽기 좋은 마크다운으로 변환하는 법

지난 편에서는 Sequential Thinking MCP를 GitHub, Notion과 조합해 설계부터 구현, 문서화까지 자동화하는 파이프라인을 구축했습니다.

이번 편에서는 새로운 MCP를 추가합니다. **웹 페이지를 AI가 바로 소비할 수 있는 마크다운으로 변환해주는 Firecrawl MCP**입니다.

---

## Firecrawl이란 무엇인가

**Firecrawl**은 웹 페이지를 LLM(대규모 언어 모델)이 읽기 좋은 형태로 변환해주는 웹 데이터 API입니다. GitHub Stars 102K를 기록한 오픈소스 프로젝트이기도 합니다.

Firecrawl을 처음 접하면 "Playwright MCP랑 뭐가 다르지?"라는 의문이 자연스럽게 생깁니다. 두 도구는 모두 웹과 상호작용하지만 설계 목적이 다릅니다.

| 항목 | Playwright MCP | Firecrawl MCP |
|------|---------------|---------------|
| 방식 | 실제 브라우저 실행 | API 기반 크롤링 |
| 강점 | 로그인, 클릭 등 UI 조작 | 빠른 대량 수집, 마크다운 변환 |
| 출력 | HTML, 스크린샷 등 | 정제된 마크다운 |
| 주 용도 | UI 자동화, 동적 페이지 | 문서화, RAG 데이터 수집 |
| 인증 | 불필요 | API 키 필요 |

Playwright는 실제 브라우저를 띄워 사람이 하는 것처럼 클릭하고 입력합니다. 반면 Firecrawl은 자체 서버가 대신 크롤링해서 **정제된 마크다운**으로 결과를 돌려줍니다.

### 마크다운 변환이 왜 AI에게 중요한가

웹 페이지는 원래 HTML 태그로 구성되어 있습니다. `<div>`, `<span>`, `<script>` 같은 태그가 실제 콘텐츠와 뒤섞여 있어 AI가 핵심 내용을 파악하기 어렵습니다. Firecrawl은 이 HTML을 `# 제목`, `**굵게**`, `- 목록` 같은 간결한 마크다운 문법으로 변환합니다. 노이즈가 제거된 텍스트이기 때문에 AI가 바로 소비하기 적합한 형태가 됩니다.

---

## API 키 발급 및 MCP 등록

### 1단계 — API 키 발급

[firecrawl.dev](https://firecrawl.dev)에 접속해 회원가입합니다. 무료 플랜으로 500크레딧이 즉시 지급됩니다. 단순 scrape 1회에 1크레딧이 소모되므로 실습과 테스트에 충분한 양입니다.

발급받은 키를 환경변수로 등록합니다.

```bash
echo 'export FIRECRAWL_API_KEY="fc-여기에_키_입력"' >> ~/.zshrc
source ~/.zshrc
```

### 2단계 — MCP 등록

```bash
claude mcp add firecrawl-mcp --scope user \
  -e FIRECRAWL_API_KEY=$FIRECRAWL_API_KEY \
  -- npx -y firecrawl-mcp
```

`--scope user`는 이 MCP를 모든 프로젝트에서 공통으로 사용하겠다는 설정입니다. 등록이 완료됐는지 확인합니다.

```bash
cat ~/.claude.json | python3 -c "import json,sys; d=json.load(sys.stdin); print(json.dumps(d.get('mcpServers',{}), indent=2))"
```

`firecrawl-mcp` 항목이 보이면 등록 완료입니다.

---

## 실습 — 3가지 핵심 도구

Firecrawl MCP는 크게 3가지 도구를 제공합니다. 각 도구의 특성과 적합한 사용 시점을 파악하는 것이 핵심입니다.

### firecrawl_scrape — 단일 페이지 변환 (1크레딧)

가장 기본적인 도구입니다. URL 하나를 마크다운으로 변환합니다.

```
"firecrawl로 https://firecrawl.dev 를 스크랩해서 마크다운으로 보여줘"
```

Claude Code가 `firecrawl_scrape`를 호출하면 약 33초 후 HTML 태그 없이 정제된 마크다운이 반환됩니다. 특정 페이지의 내용을 AI에게 빠르게 전달할 때 가장 적합합니다.

### firecrawl_crawl — 재귀 크롤링 (페이지 수만큼 크레딧)

사이트 전체를 재귀적으로 탐색합니다. 페이지 수만큼 크레딧이 소모됩니다.

```
"firecrawl로 https://docs.firecrawl.dev 를 크롤링해줘. 최대 5페이지만 수집하고 제목과 URL 목록 보여줘"
```

여기서 한 가지 주의할 점이 있습니다. 다국어를 지원하는 사이트는 `/ja/`, `/fr/`, `/es/` 같은 경로를 모두 재귀 탐색합니다. 영어 문서를 수집하려 했는데 일본어, 프랑스어 페이지가 섞여 들어오는 상황이 발생합니다. 이 경우 `includePaths` 옵션으로 수집할 경로를 명시적으로 지정해야 합니다.

```
"includePaths를 /docs/로 제한해서 영어 문서만 크롤링해줘"
```

### firecrawl_map — URL 구조 파악 (1크레딧)

사이트의 URL 목록만 빠르게 파악합니다. 실제 페이지 내용은 수집하지 않기 때문에 **단 1크레딧**으로 전체 구조를 조회할 수 있습니다.

```
"firecrawl로 https://docs.firecrawl.dev 의 URL 구조를 map으로 파악해줘"
```

결과로 총 890개의 URL과 영어 기준 326개의 구조가 한눈에 정리됐습니다.

### map → crawl 패턴

실전에서 크레딧을 절약하는 핵심 패턴입니다. **먼저 map으로 전체 URL 구조를 1크레딧에 파악한 뒤, 필요한 섹션만 선별해서 crawl**합니다. 890개 URL 전체를 무작정 crawl하는 대신 필요한 30개 페이지만 선택하면 크레딧을 90% 이상 절약할 수 있습니다.

---

## 콤보 실습(AI 뉴스 모니터링 파이프라인)

단일 도구 실습을 마친 뒤, Firecrawl을 포함한 4 MCP 콤보 실습을 진행했습니다.

**조합: Firecrawl + GitHub + Notion + Sequential Thinking**

각 MCP의 역할 분담은 다음과 같습니다.

- **Firecrawl**: Anthropic, OpenAI, HuggingFace 블로그에서 최신 뉴스 스크랩
- **Sequential Thinking**: 수집된 뉴스를 카테고리별로 신중하게 분류
- **GitHub**: 날짜별 마크다운 파일로 커밋하여 아카이빙
- **Notion**: 대시보드 페이지와 날짜별 서브페이지 자동 생성

`claude --dangerously-skip-permissions` 실행 후 프롬프트 한 줄을 입력했습니다.

전체 실행 흐름은 다음과 같습니다.

1. GitHub에 `ai-news-monitor` 레포 생성 및 README 초기 커밋
2. 3개 소스(Anthropic, OpenAI, HuggingFace)에서 총 44건의 뉴스 스크랩
3. Sequential Thinking이 각 뉴스를 **모델/기술**, **비즈니스/산업**, **연구/논문** 3개 카테고리로 분류
4. `posts/2026-04-01.md` 파일로 GitHub 커밋
5. Notion에 "AI 뉴스 모니터링" 메인 페이지와 날짜별 서브페이지 생성

Sequential Thinking의 역할이 특히 인상적이었습니다. 단순히 키워드 매칭으로 분류하는 것이 아니라 뉴스의 맥락을 파악하고 카테고리 판단 근거를 단계별로 기록하며 처리했습니다. 모호한 뉴스를 어느 카테고리에 배치할지 신중하게 판단하는 과정이 로그에 고스란히 남았습니다.

---

## 실무 활용 시나리오

Firecrawl MCP가 실무에서 유용한 대표적인 시나리오를 정리합니다.

**1. RAG 파이프라인 구축** — RAG(Retrieval-Augmented Generation)는 AI가 답변할 때 외부 문서를 검색해서 참조하는 방식입니다. Firecrawl로 공식 문서 사이트를 크롤링해 마크다운으로 수집한 뒤 벡터DB에 저장하면, AI 챗봇이 최신 문서를 실시간으로 참조할 수 있습니다.

**2. 콘텐츠 자동화** — 뉴스 사이트에서 최신 기사를 수집하고 요약해 카드뉴스 초안을 자동 생성하는 파이프라인을 구성할 수 있습니다.

**3. 경쟁사 모니터링** — 경쟁사의 문서 사이트를 map으로 구조를 파악한 뒤, 변경이 감지된 페이지만 crawl해 Notion에 기록하는 방식으로 업데이트를 추적할 수 있습니다.

**4. 외부 라이브러리 문서 스냅샷** — 자주 참조하는 라이브러리 공식 문서를 날짜별로 GitHub에 아카이빙하면 버전별 문서 변경 이력을 관리할 수 있습니다.

---

## 마무리

이번 실습으로 Firecrawl MCP의 3가지 핵심 도구(scrape, crawl, map)를 익히고, 4 MCP 콤보로 실제 동작하는 AI 뉴스 모니터링 파이프라인을 구축했습니다.

지금까지 구축한 MCP 생태계를 돌아보면 각 도구가 빈틈 없이 맞물리고 있습니다. Context7은 최신 문서를 참조하고, GitHub은 결과물을 저장하며, Notion은 대시보드를 만들고, Playwright는 브라우저를 조작하고, Supabase는 DB를 관리하고, Sequential Thinking은 복잡한 판단을 처리하며, Firecrawl은 웹 데이터를 수집합니다.

그런데 한 가지 아쉬운 점이 있습니다. 지금까지 수집하고 저장한 모든 정보는 휘발됩니다. Claude Code는 대화가 끝나면 맥락을 기억하지 못합니다. 뉴스를 수집하고 분류했지만, 다음 세션에서는 "어제 어떤 뉴스를 봤는지"를 Claude가 스스로 알 방법이 없습니다.

다음 편에서는 이 문제를 정면으로 다룹니다. **Memory MCP(Knowledge Graph)**를 연결해 Claude가 대화를 넘어 지식을 누적하고 관계를 추론할 수 있도록 만들겠습니다. 예를 들어 AI 뉴스 모니터링 파이프라인과 결합하면, 단순히 뉴스를 수집하는 것을 넘어 "지난주 GPT-5 관련 뉴스와 이번 주 뉴스 사이의 흐름"을 스스로 파악하는 에이전트로 발전시킬 수 있습니다.

---
