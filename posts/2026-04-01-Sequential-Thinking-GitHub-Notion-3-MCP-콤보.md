---
title: "[ Sequential Thinking + GitHub + Notion 3 MCP 콤보 ]"
date: 2026-04-01
tags: [MCP, SequentialThinking, claudecode, github, notion]
url: https://velog.io/@xorms/Sequential-Thinking-GitHub-Notion-3-MCP-콤보
---

# Sequential Thinking + GitHub + Notion 3 MCP 콤보 — 설계부터 구현, 문서화까지 자동화

## 들어가며

지난 편에서 Sequential Thinking MCP를 단독으로 연결해 AI가 7단계에 걸쳐 기술 의사결정을 구조적으로 추론하는 과정을 실습했습니다. 이번 편에서는 여기에 GitHub MCP와 Notion MCP를 더합니다.

콤보로 달라지는 것은 명확합니다. Sequential Thinking이 "어떻게 만들지" 설계하고, GitHub MCP가 설계한 대로 실제 코드를 커밋하고, Notion MCP가 "무엇을 왜 만들었는지" 자동으로 문서화합니다. **사람이 하는 일은 프롬프트 입력 단 한 번입니다.**

---

## 콤보 아키텍처

이번 실습의 시나리오는 Node.js + Express + JWT 기반 사용자 인증 모듈 설계, 구현, 문서화입니다.

구현할 API는 세 가지입니다.

- `POST /auth/register` — 회원가입
- `POST /auth/login` — 로그인 + JWT 발급
- `GET /auth/me` — 내 정보 조회 (JWT 검증)

파이프라인 흐름은 다음과 같습니다.

```
프롬프트 입력
→ Sequential Thinking: 어떻게 만들지 단계적으로 설계
→ GitHub MCP: 설계한 대로 실제 코드 파일 커밋
→ Notion MCP: 무엇을 왜 만들었는지 자동 문서화
```

GitHub 커밋과 Notion 페이지 생성이 연속으로 이루어지므로 `--dangerously-skip-permissions` 플래그로 실행했습니다. 중간 승인 없이 파이프라인을 한 번에 처리하기 위함입니다.

```bash
claude --dangerously-skip-permissions
```

---

## Sequential Thinking 설계 단계

프롬프트를 입력하자 Sequential Thinking이 6단계에 걸쳐 설계를 전개했습니다.

**Step 1 — 요구사항 분석**: 기술 스택을 선정했습니다. Express, JWT, bcrypt, dotenv 조합으로 결정하고 각 라이브러리를 선택한 이유를 명시했습니다.

**Step 2 — 파일 구조 설계**: routes / controllers / services / middleware 레이어를 분리하는 구조를 확정했습니다. 단순히 파일을 나누는 것이 아니라 각 레이어의 책임 범위를 명시했습니다.

**Step 3 — API 명세 설계**: 각 엔드포인트의 Request/Response 형식과 입력값 검증 규칙을 정의했습니다.

**Step 4 — 보안 고려사항(핵심 하이라이트)**

지시하지 않은 내용이 등장했습니다. Claude가 스스로 아래 두 가지를 설계에 포함했습니다.

- `bcrypt saltRounds=10` 설정
- **user enumeration 방지** — "이메일이 존재하지 않습니다"와 "비밀번호가 틀렸습니다"를 구분하지 않고 동일한 에러 메시지로 반환

user enumeration은 공격자가 이메일 존재 여부를 추론할 수 있는 보안 취약점입니다. 프롬프트에 보안 요구사항을 명시하지 않았음에도 설계 단계에서 스스로 챙긴 것입니다.

**Step 5 — GitHub/Notion 구성 계획**: 커밋 전략과 Notion 문서 구조를 미리 설계했습니다.

**Step 6 — 최종 검토 및 구현 순서 확정**: 앞선 5단계를 검토하고 구현 순서를 확정했습니다.

---

## GitHub 구현 결과

설계가 완료되자 GitHub MCP가 바로 실행됐습니다.

```
chat-app-auth/
├── src/
│   ├── controllers/authController.js
│   ├── middleware/authMiddleware.js
│   ├── routes/authRoutes.js
│   ├── services/userService.js
│   └── app.js
├── .env.example
├── .gitignore
├── index.js
├── package.json
└── README.md
```

커밋은 두 번으로 나뉘었습니다.

- **커밋 1**: `chore: initialize repository` — README.md (Sequential Thinking 실습 맥락 명시)
- **커밋 2**: `feat: add JWT auth module` — 8개 파일 일괄 커밋

이슈 #1에는 Sequential Thinking 6단계 설계 과정 전체가 기록됐습니다. 코드만 있는 레포가 아니라 "왜 이렇게 설계했는지"의 히스토리가 남는 구조입니다.

---

## 실제 서버 실행 및 API 동작 확인

```bash
git clone https://github.com/park-taegeun/chat-app-auth.git
cd chat-app-auth
npm install
cp .env.example .env  # JWT_SECRET 등 환경변수 설정
node index.js
```

curl로 세 엔드포인트를 순서대로 테스트했고 전부 정상 동작을 확인했습니다. 특히 비밀번호 6자 미만 입력 시 아래 응답이 반환됐습니다.

```json
{"error": "Password must be at least 6 characters"}
```

Step 4에서 설계한 보안 검증 규칙이 실제 코드에 그대로 반영된 것입니다. 설계와 구현이 일치하는 것을 코드 레벨에서 확인한 순간입니다.

---

## HTML 테스트 UI + Playwright 캡처 + GIF 녹화

세 번째 추가 실습으로 Sequential Thinking + GitHub + Playwright를 연결해 테스트 UI를 만들고 동작을 캡처했습니다.

### Sequential Thinking의 CORS 사전 예측

HTML UI 설계 시 흥미로운 장면이 나왔습니다. `file://` 프로토콜로 HTML을 열면 CORS 제약이 발생한다는 것을 설계 Step 1에서 스스로 발견하고, `express.static` 추가를 해결책으로 설계에 포함했습니다. 실행하기 전에 문제를 예측하고 해결책까지 설계에 반영한 것입니다.

### 결과물

- 초기 화면: 다크 테마 3카드 레이아웃, 헤더에 "● 토큰 없음" 배지 표시
- 회원가입 완료: HTTP 201 + userId 반환
- 로그인 완료: 헤더 배지가 "● 토큰 저장됨"으로 실시간 변경 + JWT 자동 저장
- 내 정보 조회: HTTP 200 + id/email/createdAt 반환
- 회원가입 → 로그인 → 내 정보 조회 전체 흐름 (960×540, 8프레임, 13.0초, 618KB)

---

AI가 코드를 만들고, AI가 UI를 테스트하고, AI가 그 장면을 GIF로 녹화했습니다. 사람의 개입은 프롬프트 입력뿐이었습니다.

---

## 핵심 인사이트

**1. 에이전트 vs 자동화 스크립트**

스크립트 자동화는 정해진 순서대로 실행할 뿐입니다. Sequential Thinking을 쓴 에이전트는 user enumeration 방지처럼 시키지도 않은 보안 고려사항을 설계 단계에서 스스로 챙겼습니다. 이것이 에이전트와 자동화 스크립트의 근본적인 차이입니다.

**2. 3분 55초의 의미**

설계 6단계 → 코드 8파일 → 이슈 등록 → Notion 문서화까지, 사람이 한 건 프롬프트 입력 단 한 번입니다.

**3. 실행 전 문제 예측**

CORS 문제를 코드를 실행하기 전 설계 단계에서 스스로 발견하고 해결책을 포함시켰습니다. Sequential Thinking의 핵심 가치는 답을 빠르게 내는 것이 아니라, 문제를 미리 발견하는 것입니다.

---

## 마무리 및 다음 편 예고

Sequential Thinking + GitHub + Notion 콤보를 완성하고 나면 개발 사이클의 구조가 달라집니다. 설계, 구현, 테스트, 문서화가 프롬프트 한 줄로 이어지고, AI가 시키지 않은 보안 고려사항까지 스스로 챙깁니다.

다음 편에서는 **Firecrawl MCP**를 다룰 예정입니다. 웹 페이지를 단순히 탐색하는 것을 넘어 구조화된 데이터로 추출하고 분석하는 워크플로우를 실습합니다.
