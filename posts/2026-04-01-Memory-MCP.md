# [ Memory MCP ]

> **시리즈**: AI 생태계 탐험 로그 11/11  
> **날짜**: 2026-04-01  
> **태그**: AI자동화, KnowledgeGraph, MCP, claudecode, memory  
> **원문**: https://velog.io/@xorms/Memory-MCP

---

## Claude Code로 Memory MCP 연결하기 — AI에게 장기 기억을 심는 법

지난 편에서는 Firecrawl MCP로 웹 페이지를 마크다운으로 변환하고, 4 MCP 콤보로 AI 뉴스 모니터링 파이프라인을 구축했습니다. 그런데 파이프라인을 완성하고 나서 한 가지 근본적인 한계가 눈에 들어왔습니다. **Claude Code는 대화가 끝나면 모든 것을 잊습니다.**

세션이 끊기면 어떤 실습을 완료했는지, 지금 어떤 프로젝트를 진행 중인지, 어떤 결정을 내렸는지 모두 사라집니다. 매번 새 세션을 시작할 때마다 배경 설명을 반복해야 하는 비효율이 생깁니다.

이번 편에서는 이 문제를 정면으로 해결합니다. **Memory MCP를 연결해 Claude에게 세션을 넘나드는 장기 기억을 부여합니다.**

---

## Memory MCP란 무엇인가

**Memory MCP**는 Anthropic이 공식 제공하는 `@modelcontextprotocol/server-memory` 패키지입니다. Claude가 정보를 로컬 파일(`memory.json`)에 저장하고, 다음 세션에서 그 기억을 불러올 수 있게 해줍니다.

단순히 텍스트를 저장하는 것이 아닙니다. **Knowledge Graph(지식 그래프)** 형태로 정보를 구조화합니다.

### Knowledge Graph란

Knowledge Graph는 **엔티티(Entity, 노드)**와 **관계(Relation, 엣지)**로 정보를 표현하는 방식입니다.

```
[박태근] --owns--> [AI생태계탐험로그]
[AI생태계탐험로그] --next_practice--> [Memory_MCP]
[박태근] --currently_learning--> [Memory_MCP]
```

노드와 엣지로 연결된 구조이기 때문에 특정 엔티티를 기준으로 관련된 모든 정보를 빠르게 조회하고, 맥락 있는 추론이 가능합니다.

### 다른 MCP들과의 역할 비교

| MCP | 역할 |
|-----|------|
| Context7 | 외부 라이브러리 공식 문서 참조 |
| Supabase | 실제 애플리케이션 DB 데이터 읽기/쓰기 |
| Memory | Claude 자신의 작업 맥락과 기억 저장/조회 |

Supabase는 애플리케이션 데이터를 다루는 도구이고, Memory는 **Claude가 스스로를 위해 사용하는 기억 공간**입니다.

---

## MCP 등록

```bash
claude mcp add --scope user memory -- npx -y @modelcontextprotocol/server-memory
```

등록 후 확인합니다.

```bash
claude mcp list
```

`memory ✓ Connected` 상태가 보이면 준비 완료입니다.

---

## 실습(Knowledge Graph 구축)

### 저장 내용 설계 원칙

Memory MCP의 목적은 **새 세션에서 컨텍스트를 빠르게 복원**하는 것입니다.

| 포함 O | 포함 X |
|--------|--------|
| 완료된 실습 목록 (이름만) | 포스팅 제목 전체 |
| 현재 진행 중인 실습 | 패키지명, 설치 명령어 |
| 프로젝트 현재 상태 | 상세 환경 정보 |

포스팅 제목 전체나 패키지명까지 저장하면 그래프가 불필요하게 커집니다. 핵심 맥락만 선별해서 넣는 것이 올바른 설계입니다.

### 실제 저장 프롬프트

```
Memory MCP를 사용해서 다음 정보를 Knowledge Graph로 저장해줘.

엔티티:
- 박태근 (person)
- AI생태계탐험로그 (project)
- Memory_MCP (tool)
- 졸업작품 (project)
- velog_backup (infrastructure)

관계:
- 박태근 --owns--> AI생태계탐험로그
- 박태근 --currently_learning--> Memory_MCP
- AI생태계탐험로그 --next_practice--> Memory_MCP
- 박태근 --working_on--> 졸업작품
- AI생태계탐험로그 --uses--> velog_backup
```

`create_entities`로 노드 5개, `create_relations`로 관계 6개가 순서대로 생성됐습니다.

### 새 세션 조회 테스트

Claude Code를 완전히 종료한 뒤 재실행했습니다. 새 세션에서 아래 프롬프트를 입력했습니다.

```
Memory MCP에서 박태근 관련 정보를 모두 조회해줘
```

저장했던 엔티티와 관계가 전부 정상적으로 복원됐습니다. 새 세션임에도 이전 작업 맥락을 바탕으로 바로 이어서 대화할 수 있었습니다.

---

## 주의할 점 — 트레이드오프

**1. 수동 유지보수 부담**  
Claude가 자동으로 기억을 업데이트해주지 않습니다. 새 실습을 완료하거나 프로젝트 상태가 바뀌면 직접 갱신 요청을 해야 합니다. 해결책은 **기존 Velog 동기화 루틴에 Memory 업데이트를 끼워 넣는 것**입니다.

**2. 명시적 호출 필요**  
Memory MCP를 연결했다고 해서 세션 시작 시 자동으로 읽히지 않습니다. 새 세션에서 기억을 활용하려면 명시적으로 조회를 요청해야 합니다. 세션 시작 시 "Memory MCP에서 현재 프로젝트 상태 조회해줘"를 습관적으로 입력하는 것이 좋습니다.

**3. 로컬 파일 의존**  
Knowledge Graph는 로컬의 `memory.json` 파일에 저장됩니다. 개발 환경을 새 기기로 이전할 경우 이 파일을 직접 복사해야 합니다. 클라우드 동기화는 기본 제공되지 않습니다.

---

## 실무 활용 시나리오

**1. 장기 프로젝트 컨텍스트 유지**  
세션마다 배경 설명을 반복할 필요가 없습니다. 새 세션에서 Memory를 조회하면 바로 이어서 작업할 수 있습니다.

**2. 사용자 프로필 누적**  
선호하는 기술 스택, 작업 스타일, 완료된 실습 목록을 엔티티로 저장해두면 Claude가 맥락에 맞는 제안을 할 수 있습니다.

**3. 의사결정 기록**  
기술 선택 이유, 트레이드오프 판단 내역을 관계로 저장해두면 나중에 "왜 이 방식을 선택했는지"를 빠르게 추적할 수 있습니다.

---

## 마무리

이번 실습으로 Memory MCP를 연결하고 Knowledge Graph 형태로 프로젝트 컨텍스트를 저장해 세션을 넘나드는 기억을 구현했습니다.

지금까지 구축한 MCP 생태계가 이제 한층 완성에 가까워졌습니다. 문서를 참조하고, 코드를 저장하고, 대시보드를 만들고, 브라우저를 조작하고, 데이터를 수집하고, 복잡한 판단을 처리하는 것에 더해 이제 Claude가 **스스로의 작업 맥락을 기억**할 수 있게 됐습니다.

다음 편에서는 Memory MCP를 기존 파이프라인과 결합합니다. **Memory + Playwright + Notion 콤보**로 세션 종료 시 Velog 크롤링, GitHub 백업, Notion 업데이트, Memory 갱신이 한 번의 프롬프트로 동시에 처리되는 완전 자동화 루틴을 구축할 예정입니다.
