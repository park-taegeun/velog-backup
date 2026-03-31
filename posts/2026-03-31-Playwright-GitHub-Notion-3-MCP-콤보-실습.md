---
title: "[ Playwright + GitHub + Notion 3 MCP 콤보 실습 ]"
date: 2026-03-31
tags: [MCP, claudecode, github, notion, playwright]
original_url: https://velog.io/@xorms/Playwright-GitHub-Notion-3-MCP-콤보-실습
---

# Playwright + GitHub + Notion 3 MCP 콤보 — Velog 크롤링부터 대시보드까지 자동화

## 들어가며

지금까지 총 4개의 MCP를 연결했습니다.

Context7으로 공식 문서를 실시간 참조하고, GitHub MCP로 브랜치와 PR을 자연어로 관리하고, Notion MCP로 워크스페이스 문서를 자동으로 생성하고, Playwright MCP로 실제 브라우저를 직접 조작했습니다.

이번 편은 이 MCP들을 처음으로 **동시에** 사용합니다.

Playwright, GitHub, Notion — 세 MCP가 하나의 프롬프트 안에서 협력해 Velog 전체 포스팅을 크롤링하고, GitHub에 마크다운으로 백업하고, Notion 대시보드까지 자동 생성하는 풀 파이프라인을 구성합니다.

---

## 3 MCP 콤보 구조

### 역할 분담

| MCP | 담당 역할 |
| --- | --- |
| Playwright | 실제 브라우저로 Velog 페이지 접속 및 포스팅 본문/태그/날짜 크롤링 |
| GitHub | velog-backup 레포 생성 + 마크다운 파일 커밋 + README 자동 생성 |
| Notion | 포스팅 대시보드 페이지 생성 + 날짜/제목/태그/링크 표로 정리 |

### 에이전트적 자율 실행 흐름

MCP 콤보의 핵심은 개발자가 "어떤 도구를 언제 쓸지" 직접 지시하지 않아도 된다는 점입니다. 프롬프트 한 줄을 입력하면 Claude Code가 목표 달성을 위한 최적의 도구 순서를 스스로 결정합니다.

이번 실습에서 Claude Code가 자율적으로 수행한 흐름은 다음과 같습니다.

```
Playwright로 포스팅 목록 페이지 접속
→ JavaScript 실행으로 포스팅 카드 전체 파싱
→ 각 포스팅 URL 순회하며 본문/태그/날짜 추출
→ GitHub MCP로 velog-backup 레포 생성
→ 포스팅별 마크다운 파일 개별 커밋 (총 11 commits)
→ README.md에 날짜/제목/태그 표 자동 생성
→ Notion MCP로 대시보드 페이지 생성 + 표 정리
```

단일 프롬프트 입력 후 7단계를 사용자 개입 없이 자율 처리했습니다.

---

## --dangerously-skip-permissions 플래그

Claude Code는 기본적으로 도구 호출마다 사용자 승인을 요청합니다.

10개 포스팅 크롤링과 GitHub 커밋, Notion 기록처럼 반복 호출이 많은 작업에서는 매번 승인하는 것이 비효율적입니다.

이럴 때 `--dangerously-skip-permissions` 플래그를 사용합니다.

```
claude --dangerously-skip-permissions
```

영구 설정이 아닌 세션 단위 적용이며, 종료 후 다시 `claude`로 실행하면 승인 모드로 돌아옵니다.

**써도 되는 상황:** 읽기 전용 크롤링, 백업처럼 되돌리기 쉬운 반복 작업

**쓰면 안 되는 상황:** 프로덕션 DB 수정, 배포 자동화처럼 되돌리기 어려운 작업

---

## 실습 과정(작성자 환경)

### 1단계: 실행

```
claude --dangerously-skip-permissions
# 경고 메시지 확인 후 "Yes, I accept" 선택
```

### 2단계: 3 MCP 콤보 프롬프트 입력

```
내 Velog(https://velog.io/@xorms/posts)의 모든 포스팅을
Playwright로 크롤링해서 각 글의 제목, 날짜, 태그, 본문을 추출하고
GitHub에 velog-backup 레포를 만들어서 각 포스팅을 마크다운 파일로 저장해줘.
posts/ 폴더에 날짜-제목.md 형식으로 커밋하고
README.md에 포스팅 목록도 자동으로 정리해줘.
```

### 3단계: Slithering... + /btw로 진행 상황 확인

프롬프트 입력 후 Claude Code는 **Slithering...** 로딩 상태로 진입합니다.

뱀이 미끄러지듯 움직이는 것을 연상한 네이밍으로, 장시간 작업 중임을 나타냅니다.

작업을 중단하지 않고 진행 상황을 확인하려면 `/btw` 명령어를 사용합니다.

```
/btw 얼마나 남았어?
```

현재까지 완료된 포스팅 목록과 남은 작업량을 실시간으로 확인할 수 있습니다. 이번 실습에서는 총 **20분 9초** 동안 작업이 진행됐고, 중간에 7/10 완료 상태를 확인하는 데 활용했습니다.

### 4단계: Notion 대시보드 생성

GitHub 백업이 완료된 후 Notion 대시보드를 별도 프롬프트로 요청했습니다.

```
내 Notion에 'Velog 포스팅 대시보드'라는 페이지를 만들고
velog-backup 레포에 있는 포스팅 목록(제목, 날짜, 태그, Velog 링크)을
표로 정리해줘
```

→ Notion MCP로 대시보드 페이지 자동 생성 완료

---

## 결과물 확인

### GitHub velog-backup 레포

```
posts/
├── 2026-03-30-Playwright-MCP-Server.md
├── 2026-03-30-Notion-MCP.md
├── 2026-03-27-Context7-GitHub-MCP-콤보.md
├── 2026-03-27-GitHub-MCP.md
├── 2026-03-27-Context7-MCP.md
└── (총 10개)
README.md (날짜/제목/태그 표 자동 생성)
```

포스팅별 개별 커밋(총 11 commits)으로 히스토리가 남고, README에는 날짜/제목/태그 표가 자동 생성됐습니다.

레포 설명도 "xorms.log Velog 포스팅 백업 — Playwright로 자동 크롤링한 마크다운 아카이브"로 자동 입력됐습니다.

### Notion 대시보드

"Velog 포스팅 대시보드" 페이지에 날짜 / 제목 / 태그 / Velog 링크 / GitHub 링크가 표 형태로 정리됐습니다.

---

## 트러블슈팅

### 졸업작품 시리즈 제외 실패 → 교훈: 조건은 처음 프롬프트에

크롤링이 시작된 후 "졸업작품 기록 시리즈는 제외해줘"라는 조건을 추가하려 했으나, 이미 작업이 진행 중이어서 전체 포함으로 완료됐습니다. 중간에 조건을 추가하는 것은 적용되지 않습니다.

처음 프롬프트에 조건을 명시하는 것이 올바른 방식입니다.

```
단, '졸업작품 기록' 시리즈는 제외해줘  ← 최초 프롬프트에 포함해야 함
```

---

## 실무 확장 가능성

**최신화 동기화 운영**

새 포스팅 작성 후 아래 프롬프트 한 줄로 GitHub + Notion을 동시에 업데이트할 수 있습니다.

```
velog-backup 최신화해줘
```

**GitHub Actions 완전 자동화**

현재 구조는 프롬프트 실행 시점에 동기화되는 방식입니다.

매일 자정 자동 실행 스케줄러를 GitHub Actions로 구성하면 Claude Code 없이 Playwright 코드만으로 완전 자동화가 가능합니다.

**졸업 프로젝트 CI/CD 파이프라인**

Playwright(UI 테스트) + GitHub(이슈 자동 생성) + Notion(테스트 결과 기록)을 연결하면 "테스트 실패 → 스크린샷 캡처 → 이슈 생성 → Notion 기록"까지 풀 사이클 자동화가 가능합니다.

---

## 마무리 및 다음 편 예고

3 MCP 콤보 실습을 마치고 나면 에이전트 워크플로우가 어떤 의미인지 체감할 수 있습니다. 도구를 하나씩 연결하는 것과, 여러 도구가 하나의 목표를 향해 자율적으로 협력하는 것은 전혀 다른 경험입니다. 20분 동안 Slithering... 상태로 스스로 크롤링하고 커밋하고 대시보드를 만들어내는 장면은 단순한 자동화가 아닙니다.

다음 편에서는 **PostgreSQL / Supabase MCP 연동**을 다룰 예정입니다.

Claude Code에서 데이터베이스에 자연어로 직접 쿼리하고, 테이블 생성부터 데이터 조회, 수정까지 처리하는 워크플로우를 실습합니다.

---
