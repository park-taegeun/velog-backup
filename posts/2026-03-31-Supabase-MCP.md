---
title: "[ Supabase MCP ]"
date: "2026-03-31"
tags: ["AI에이전트", "MCP", "PostgreSQL", "claudecode", "supabase"]
series: "AI 생태계 탐험 로그"
velog_url: "https://velog.io/@xorms/Supabase-MCP"
---

# Claude Code로 Supabase MCP 연결하기 — 자연어로 DB 설계부터 데이터 조작까지

## 들어가며

지난 편에서 Playwright, GitHub, Notion MCP를 동시에 연결해 Velog 포스팅 크롤링부터 GitHub 백업, Notion 대시보드 생성까지 프롬프트 한 줄로 처리하는 풀 파이프라인을 완성했습니다. 이번 편에서는 여기에 **데이터베이스**를 추가합니다.

Supabase MCP를 연결하면 달라지는 것은 하나입니다. SQL을 직접 작성하지 않아도 자연어만으로 DB 스키마를 설계하고, 데이터를 조작하고, 보안 정책까지 적용할 수 있게 됩니다.

---

## Supabase란 무엇인가

Supabase는 PostgreSQL을 기반으로 한 BaaS(Backend as a Service)입니다. DB 설치와 설정 없이 클라우드에서 바로 PostgreSQL을 사용할 수 있으며, REST API 자동 생성, Auth, Storage, Realtime 기능을 함께 제공합니다. Firebase의 오픈소스 대안으로 불리며, 백엔드 인프라를 직접 구성하지 않아도 빠르게 서비스를 프로토타이핑할 수 있다는 점에서 사이드 프로젝트와 졸업 프로젝트에 자주 활용됩니다.

---

## Supabase MCP란 무엇인가

Supabase MCP는 Claude Code가 내 Supabase 프로젝트에 직접 접근해 자연어 명령만으로 DB를 설계하고 조작할 수 있게 해주는 MCP 서버입니다. 내부 동작 흐름은 다음과 같습니다.

```
자연어 입력 → Claude가 SQL로 변환 → Supabase MCP 도구 호출 → PostgreSQL DB 실행
```

### 인증 방식 변화 — PAT에서 OAuth로

과거에는 PAT(Personal Access Token)를 헤더에 직접 넣는 방식이었으나, 현재는 Notion MCP와 동일하게 **OAuth 브라우저 인증 방식**으로 변경되었습니다. `/mcp` 명령어로 브라우저 로그인 후 자동 연결되며, 토큰을 직접 관리할 필요가 없어 보안 측면에서도 유리합니다.

---

## 연결 실습(작성자 환경)

### 1단계: Supabase 프로젝트 생성

[supabase.com](https://supabase.com) 접속 후 GitHub 로그인 → New project 선택

- 이름: `mcp_practice`
- Region: `Northeast Asia (Seoul)`

`ACTIVE_HEALTHY` 상태가 되면 준비 완료입니다.

### 2단계: Supabase MCP 등록

```bash
claude mcp add --transport http supabase "https://mcp.supabase.com/mcp" --scope user
# Added HTTP MCP server supabase 메시지 확인
```

### 3단계: Claude Code 실행 및 OAuth 인증

```bash
claude
```

`/mcp` 입력 → `supabase · △ needs authentication` 확인 → `supabase` 선택 후 Enter → 브라우저 OAuth 로그인

```
Authentication successful. Connected to supabase.
```

### 4단계: 프로젝트 조회 테스트

```
내 Supabase 프로젝트 목록 보여줘. MCP 툴 사용해서.
```

→ `mcp_practice` 프로젝트의 이름, ID, 리전, 상태, DB 버전, 생성일이 정상 조회됩니다.

---

## 실습(스키마 설계)

블로그 서비스를 가정하고 테이블 3개를 설계했습니다.

| 테이블 | 주요 컬럼 | 관계 |
|--------|-----------|------|
| users | id, email, name, created_at | — |
| posts | id, user_id(FK), title, content, published, created_at | users 참조 |
| comments | id, post_id(FK), user_id(FK), body, created_at | posts, users 참조 |

자연어로 테이블 구조를 설명하면 Claude가 SQL로 변환해 Supabase에 즉시 실행합니다. FK(Foreign Key) 설정과 `ON DELETE CASCADE`도 자동으로 적용됩니다. 테이블 생성 후 Supabase Schema Visualizer에서 관계도를 시각적으로 확인할 수 있습니다.

---

## 실습(데이터 CRUD)

테스트 데이터를 삽입하고 다양한 쿼리를 실습했습니다.

- **INSERT**: users 3명, posts 6개(유저당 2개), comments 12개(포스트당 2개) 삽입
- **SELECT + JOIN**: published 포스트와 작성자 이름을 함께 조회
- **GROUP BY**: 댓글 수 기준으로 포스트 정렬
- **UPDATE**: 특정 포스트의 published 상태 변경
- **DELETE**: 특정 유저의 댓글 전체 삭제

복잡한 JOIN 쿼리도 "published된 포스트를 작성자 이름과 함께 보여줘"처럼 자연어로 지시하면 Claude가 SQL을 직접 작성해 실행합니다.

---

## 실습(RLS)

RLS(Row Level Security)는 PostgreSQL의 행 단위 보안 기능입니다. RLS를 활성화하면 테이블에 접근 정책(Policy)을 설정할 수 있으며, DB 레벨에서 접근 규칙을 강제할 수 있습니다.

`posts` 테이블에 아래 3가지 정책을 적용했습니다.

| 정책 | 조건 | 설명 |
|------|------|------|
| SELECT | `published = true` | 공개된 포스트만 누구나 조회 가능 |
| UPDATE | `auth.uid() = user_id` | 본인이 작성한 포스트만 수정 가능 |
| DELETE | `auth.uid() = user_id` | 본인이 작성한 포스트만 삭제 가능 |

`auth.uid()`는 현재 로그인한 유저의 ID를 반환하는 Supabase 내장 함수입니다. INSERT 정책을 별도로 설정하지 않으면 RLS 활성화 상태에서 데이터 삽입이 불가능합니다. DB 레벨에서 보안을 강제하기 때문에 애플리케이션 코드의 보안 로직을 단순화할 수 있습니다.

---

## 트러블슈팅(작성자 경험)

### PAT 방식 시도 → Failed to connect

Supabase MCP 문서 일부에 PAT 토큰을 헤더에 직접 넣는 방식이 안내되어 있어 시도했으나 연결에 실패했습니다. Supabase MCP가 OAuth 방식으로 변경된 것이 원인이었습니다.

```bash
# 실패한 방식 (구버전)
claude mcp add supabase -e SUPABASE_ACCESS_TOKEN=... -- npx ...

# 현재 올바른 방식
claude mcp add --transport http supabase "https://mcp.supabase.com/mcp" --scope user
# 이후 /mcp에서 OAuth 인증
```

---

## 실무 활용 시나리오

**1. 개발 단계별 활용**

프로토타이핑 단계에서는 "쇼핑몰 서비스에 필요한 테이블 구조 설계하고 바로 만들어줘" 한 줄로 ERD 설계부터 SQL 실행까지 단축할 수 있습니다. 개발 중 스키마 변경이 필요할 때는 "users 테이블에 profile_image 컬럼 추가해줘"처럼 즉시 반영이 가능합니다. 디버깅 시에는 "user_id가 null인 이상한 데이터 찾아줘"처럼 자연어로 이상 데이터를 탐지할 수 있습니다.

**졸업 프로젝트 MCP 풀 사이클**

지금까지 배운 MCP를 조합하면 하나의 개발 워크플로우가 완성됩니다.

| MCP | 역할 |
|-----|------|
| Context7 | 사용할 라이브러리 최신 문서 참조 |
| Supabase | DB 스키마 설계 + 테이블 생성 |
| GitHub | 코드 커밋 + PR |
| Notion | 진행 상황 자동 기록 |

---

## 마무리 및 다음 편 예고

Supabase MCP를 연결하고 나면 데이터베이스 작업의 진입 장벽이 낮아집니다. 테이블 설계부터 RLS 정책 적용까지 자연어로 처리할 수 있고, SQL을 아는 개발자라면 더 정교한 결과를 끌어낼 수 있습니다.

다음 편에서는 **Sequential Thinking MCP**를 다룰 예정입니다. 복잡한 아키텍처 설계나 디버깅 시 Claude의 사고를 구조화해, 답을 바로 내놓는 대신 문제를 단계별로 분석하고 접근법을 수정하면서 깊은 추론을 수행하는 워크플로우를 실습합니다.
