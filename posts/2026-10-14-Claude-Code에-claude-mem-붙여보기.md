# [ Claude Code에 claude-mem 붙여보기 ]

- 작성일: 2026-10-14
- 원문: https://velog.io/@xorms/Claude-Code에-claude-mem-붙여보기
- 태그: AI메모리, claude code, 플러그인

---

지난 편에서는 공식 플러그인 claude-code-setup으로 졸업작품 레포를 진단하고, hook과 permissions.deny로 비밀값 이중 방어를 구축했습니다. 이번 편에서는 같은 "필수 플러그인 5개" 가이드의 세 번째 도구, **claude-mem**을 다룹니다.

Claude Code는 세션이 끝나면 맥락을 잊습니다. 그래서 새 세션마다 "이 프로젝트는 무엇이고, 지난번에 어디까지 했는지"를 다시 설명해야 합니다. 11편에서 Memory MCP로 이 문제를 풀어봤지만, 자동으로 갱신되지 않고 매번 명시적으로 불러야 한다는 한계가 남았습니다. claude-mem은 hook으로 작업을 자동 기록해 이 한계를 정면으로 겨냥한 도구입니다.

결론부터 말하면, **"자동 기억"을 실험해 보니 실제로는 "자동 목차"였습니다.**

> 실습 환경: macOS(Apple Silicon), Claude Code v2.1.299, claude-mem v13.39.0, Node.js v24.7.0, Bun 1.4.3, uv 0.12.10

---

## claude-mem은 어떻게 기억하는가

### 5단계 동작 구조

1. **기록**: hook이 Claude의 도구 사용(파일 읽기·쓰기, 명령 실행)을 **관찰(observation)**로 포착합니다. 관찰은 "Claude가 무엇을 했는지"를 남긴 기록 한 건입니다. v13.39.0 기준 hook은 Setup, SessionStart, UserPromptSubmit, PreToolUse, PostToolUse, PostToolUseFailure, Stop, SessionEnd 8개입니다.
2. **정리**: 백그라운드 워커(화면 뒤에서 계속 도는 프로그램)가 요약 담당 AI인 **provider**로 관찰을 요약하고 유형을 분류합니다.
3. **저장**: 파일 하나로 동작하는 가벼운 DB인 SQLite(`~/.claude-mem/claude-mem.db`)와, 뜻으로 찾기 위한 Chroma 벡터 색인에 저장합니다.
4. **주입**: 다음 세션이 시작되면 최근 관찰을 맥락으로 자동 주입합니다.
5. **검색**: 상세 내용은 MCP 도구(`get_observations` 등)로 필요할 때 꺼내 봅니다.

### 점진적 공개

claude-mem의 핵심 설계는 **점진적 공개(progressive disclosure)**입니다. 처음에는 목차만 보여주고, 필요할 때만 본문을 펼치는 방식입니다. 도서관에서 책 전체를 빌리지 않고 목차부터 훑어보는 것과 같습니다. 관찰이 100개 쌓여도 매 세션 컨텍스트에는 제목만 올라가므로 토큰 부담이 크지 않습니다.

### 기억 장치 4종 비교

Claude Code에서 "기억"을 담당하는 장치는 이미 여러 개입니다. Claude Code auto memory는 Claude가 중요하다고 판단한 사실을 스스로 메모해두는 내장 기능입니다.

| | CLAUDE.md | Claude Code auto memory | Memory MCP | claude-mem |
|---|---|---|---|---|
| 누가 기록 | 사람 | Claude가 판단해서 | 명시적으로 시킬 때만 | hook이 자동으로 |
| 무엇을 | 규칙·명령어 | 중요한 사실 한 줄 요약 | 엔티티·관계 | 작업 이력 |
| 세션 시작 때 | 전체 주입 | 목록(MEMORY.md) 주입 | 주입 안 됨 | 제목 목록만 주입 |
| 추가 비용 | 없음 | 없음 | 없음 | 요약용 AI 토큰 |
| 형태 | 내장 | 내장 | MCP | 서드파티 플러그인 |

---

## 설치 전 점검: 가이드에 없던 함정 3가지

가이드를 따라 하기 전에 공식 문서를 먼저 읽었고, 가이드에 없던 함정 3가지를 찾았습니다.

1. **유료 클라우드가 기본 선택**: 가이드의 `npx claude-mem install` 대화형 설치기는 유료 클라우드 "CMEM Pro"(30일 체험 후 구독)를 기본으로 골라둡니다. 이걸 선택하면 관찰 기록이 cmem.ai 서버를 거칩니다. 그래서 `/plugin` 방식으로 설치했습니다.
2. **구독 인증이 기본 provider**: 기본 provider가 claude(구독 인증)이고, provider가 막히면 구독으로 자동 전환하는 quota fallback도 있습니다. 설정 문서와 Gemini 문서의 기본값 설명이 서로 달라서 fallback을 명시적으로 off로 지정했습니다.
3. **텔레메트리가 기본 활성**: 사용 통계를 보내는 익명 텔레메트리가 opt-out(끄지 않으면 켜짐) 방식입니다. `npx` 설치기는 동의 여부를 묻지만, `/plugin` 방식은 그 질문을 건너뜁니다.

### provider와 레포 민감도 원칙

| 선택지 | 비용 | 계정 리스크 | 데이터 행방 |
|---|---|---|---|
| Gemini 무료 키 (선택) | 무료 | 없음 | Google (무료 등급은 제품 개선에 사용될 수 있고 사람 검토 가능) |
| Anthropic API 키 | 종량제 | 없음 | Anthropic |
| 구독 (기본값) | 구독 한도 내 | 회색지대 | Anthropic |

원칙은 **provider를 레포의 민감도로 고르는 것**입니다. 공개 뉴스를 다루는 ai-news-monitor에는 Gemini 무료 키를 쓸 수 있지만, 비밀값과 미공개 설계가 있는 졸업작품 레포에는 쓰지 않습니다.

### 구독 OAuth 회색지대

Anthropic은 Free·Pro·Max 구독의 OAuth 토큰을 Claude Code·Claude.ai 이외의 도구(Agent SDK 포함)에서 쓰는 것을 허용하지 않는다고 문서에 밝혔습니다. 동시에 Agent SDK를 활용한 로컬 개발·실험은 장려한다고도 밝혔습니다. claude-mem의 기본값은 워커가 Claude Code 로그인 토큰을 읽어 자체 Agent SDK 프로세스에 넣는 구조라 이 회색지대에 걸립니다.

---

## 설치

### 1. Gemini API 키 발급과 검증

aistudio.google.com에서 키를 발급하고 Plan이 무료인지 확인했습니다. 키는 채팅창에 절대 붙여넣지 않았습니다. 발급된 키는 흔히 알려진 `AIza…` 형식이 아니라 `AQ.…` 형식이었는데, 추측하지 않고 실제 API 호출로 검증했습니다.

### 2. 설치 전에 설정 파일부터

설치 직후 첫 세션이 기본값(구독)으로 요약을 시작하지 않도록, **설정 파일을 설치보다 먼저** 만들었습니다. 키는 클립보드에서 바로 읽어 화면과 셸 기록에 남지 않게 했습니다.

```bash
python3 - <<'EOF'
import json, os, subprocess
path = os.path.expanduser("~/.claude-mem/settings.json")
if os.path.exists(path):
    raise SystemExit("이미 있음: 덮어쓰지 않음")
key = subprocess.run(["pbpaste"], capture_output=True, text=True).stdout.strip()
os.makedirs(os.path.dirname(path), exist_ok=True)
json.dump({
    "CLAUDE_MEM_PROVIDER": "gemini",
    "CLAUDE_MEM_GEMINI_API_KEY": key,
    "CLAUDE_MEM_QUOTA_FALLBACK_PROVIDER": "off",
}, open(path, "w"), indent=2)
os.chmod(path, 0o600)
print("key=" + key[:4] + "*" * (len(key) - 4))
EOF
pbcopy < /dev/null
```

기존 파일 덮어쓰기 방지, 나만 읽고 쓸 수 있는 권한(600), 앞 4자리만 보이는 마스킹 출력, 마지막 클립보드 비우기까지 넣었습니다. 검증할 때도 키를 URL이 아닌 **헤더**로 전달했습니다. URL은 로그나 기록에 남기 쉽기 때문입니다.

```bash
python3 - <<'EOF'
import json, os, urllib.request
key = json.load(open(os.path.expanduser("~/.claude-mem/settings.json")))["CLAUDE_MEM_GEMINI_API_KEY"]
req = urllib.request.Request("https://generativelanguage.googleapis.com/v1beta/models",
                             headers={"x-goog-api-key": key})
print(len(json.load(urllib.request.urlopen(req))["models"]), "models")
EOF
```

모델 50개가 조회되어 키가 정상임을 확인했습니다.

### 3. 준비물과 실습 레포

Node와 uv는 이미 있었고 Bun만 없었습니다. `/plugin` 방식은 Bun을 자동 설치하지 않습니다. 실습 레포 ai-news-monitor는 예전에 GitHub MCP로 원격에만 만든 레포라 로컬에 클론했습니다.

```bash
brew install oven-sh/bun/bun
mkdir -p ~/projects && cd ~/projects
git clone git@github.com:park-taegeun/ai-news-monitor.git
```

### 4. local scope 설치

```
/plugin marketplace add thedotmack/claude-mem
/plugin install claude-mem@thedotmack
```

관리 화면 없이 바로 설치되어 user scope로 들어갔을까 걱정했지만, `/plugin` → Installed 탭에서 **Scope: local**을 확인했습니다. 구성은 skills 24개, hooks 8개, MCP 1개(mcp-search)였습니다. 스킬 24개는 이름과 설명이 매 세션 컨텍스트를 조금씩 차지합니다.

### 5. 첫 실행 hook 오류 진단

```
SessionStart:startup hook error
Executable not found in $PATH: "claude-mem"
```

추측하지 않고 하나씩 확인했습니다.

- 설치된 13.39.0의 hooks.json은 `claude-mem` 실행 파일을 직접 호출했습니다. GitHub main 브랜치는 `node` 스크립트 방식이라 형태가 달랐습니다.
- `which -a claude-mem`으로 보니 `~/.local/bin/claude-mem`(약 70MB 단일 바이너리)은 존재했습니다.
- 파일 생성 시각 13:42가 첫 세션 시작 시각과 같았습니다.

결론은 **Setup hook이 실행 파일을 만드는 도중에 SessionStart hook이 먼저 실행된 첫 실행 타이밍 문제**였고, 재시작으로 해결됐습니다. 파일 시각이라는 증거가 원인을 확정해줬습니다.

---

## 데이터 경로 검증

### 권한 모드 정리

셸 히스토리 때문에 `--dangerously-skip-permissions`(모든 승인 단계를 건너뛰는 옵션)로 실행되던 것을 일반 실행으로 바꿨습니다. 승인 단계는 마지막 확인 버튼이고, 외부 웹을 긁는 레포라 프롬프트 인젝션 위험이 있으며, 서드파티 스킬이 24개나 되기 때문입니다. 이 옵션은 인터넷이 차단된 컨테이너 같은 격리 환경용입니다. 반복 승인이 번거롭다면 Shift+Tab으로 accept edits·auto 모드를 쓰거나 `permissions.allow` 규칙을 두는 편이 낫습니다.

### 워커가 정말 Gemini를 쓰는가

```bash
curl -s http://127.0.0.1:37701/api/health
```

provider는 gemini였지만, authMethod에는 "Claude Code OAuth token (read from the Claude Code credential store at spawn)"이 표시됐습니다. `CLAUDE_MEM_CLAUDE_AUTH_METHOD`를 `api-key`(키 없음)로 추가하고 워커를 재시작했지만 표시는 그대로였습니다(효과 여부 불명).

그래서 로그로 검증했습니다. 키를 마스킹한 뒤 grep한 결과, 요약 작업은 전부 "using Gemini"였고 Anthropic·OAuth 관련 줄은 0건이었습니다. 문서상으로도 Gemini 사용 시 관찰 요약은 구독을 쓰지 않습니다. 그대로 진행하되, Claude를 직접 부를 수 있는 스킬(`/learn-codebase`, knowledge-agent)은 쓰지 않기로 했습니다.

### 텔레메트리 끄기

```bash
npx claude-mem telemetry disable
npx claude-mem telemetry status   # Telemetry: DISABLED
```

---

## 실험: 무엇이 기억되나

### 탐침 설계

탐침(probe)은 나중에 기억 여부를 확인하려고 일부러 심어두는 정보입니다. 세션 1에 탐침 2개를 심고, 세션 2에서 기억만으로 답하게 했습니다. 두 탐침은 실험용으로 지어낸 정보입니다.

| 탐침 | 심는 방법 | 처음 가설 |
|---|---|---|
| A. "카테고리가 겹치면 모델/기술 우선" | 파일(docs/category-rules.md)에 기록 | 기억됨 |
| B. "다음 주 DeepMind 소스 추가, 미확정" | 대화로만 전달 ("파일엔 적지 마") | 불확실 |

### 혼입 변수 발견과 변인 통제

혼입 변수(confounding variable)는 실험 결과에 다른 원인이 끼어드는 것입니다. 세션 1을 마치고 보니 Claude가 탐침 B를 Claude Code 자체 auto memory(`~/.claude/projects/<project>/memory/`)에 따로 저장해 두었습니다. 이대로 세션 2를 진행하면 claude-mem이 기억한 것인지 auto memory가 기억한 것인지 구분할 수 없습니다. 메모 파일에 Claude가 쓰지 않은 메타데이터(node_type, originSessionId)가 붙어 있었는데, 출처는 확인하지 못했습니다.

그래서 세션 2 동안 auto memory 폴더를 `memory.off`로 이름을 바꿔 치워두는 방식으로 통제했습니다.

### 세션 2 — 자동 주입된 목차

```
#1 ⚖(decision) News categorization rules and sources defined
#2 ⊘(sensitive) Planned addition of Google DeepMind blog source
```

탐침 B는 "미확정, 적지 마"라는 맥락 때문에 **민감 정보(sensitive)**로 분류됐습니다.

### 테스트 1 — 도구 사용 금지, 주입된 맥락만으로 답하기

| 질문 | 결과 |
|---|---|
| A 우선 카테고리 | ❌ 모름 (제목만 보임) |
| B DeepMind 계획 존재 | ✅ |
| B "다음 주" 시점 | ❌ 모름 |
| B 미확정 여부 | △ "Planned"로 짐작 |

### 테스트 2 — `get_observations`만 허용

모든 질문에 정답을 맞혔습니다(모델/기술 우선, 다음 주, "not yet finalized"). MCP 도구 스키마는 필요할 때 ToolSearch로 불러왔고, Claude는 "기록에 근거했지만 파일로 검증하진 않았다"고 스스로 구분해서 답했습니다.

### 반전 해석

기억 여부를 가른 것은 **"파일에 썼냐, 말로 했냐"가 아니라 "관찰 제목에 들어갔냐"**였습니다. 파일에 쓴 탐침 A도 세부 내용은 제목에 없어서 테스트 1에서 몰랐고, 말로만 한 탐침 B도 계획의 존재는 제목에 들어가서 알았습니다.

claude-mem의 "자동 기억"은 정확히 말하면 **자동 목차(1층) + 필요 시 열람(2층)** 구조입니다. 목차만으로는 세부를 모르고, 본문을 열어야 정확해집니다.

덤으로 발견한 문제도 있습니다. 띵동용으로 만든 decisions-rag MCP가 user scope로 등록되어 있어 ai-news-monitor 세션에도 안내문이 들어왔고, Claude가 엉뚱한 레포 파일을 근거 후보로 언급했습니다. MCP의 설치 범위 설계가 중요한 이유입니다.

---

## 비용과 지연 분석

| 항목 | 값 |
|---|---|
| 세션 시작 시 목차 읽기 비용 (주입 화면 추정) | 335토큰 |
| 테스트 2에서 본문 열람에 쓴 추가 토큰 | 약 326토큰 |
| 원래 작업에 쓴 토큰 | 7,296토큰 |
| 관찰 기록 생성 (Gemini) | 입력 약 9.2K + 출력 약 1.5K ≈ 1만 토큰 |
| 기록 지연 | 작업 종료 13:59 → 관찰 저장 14:06, 약 7분 |

주입 화면은 "95% 절약"이라고 표시했습니다. 7,296토큰짜리 작업을 335토큰으로 요약해 보여줬다는 뜻으로, **Claude 컨텍스트 기준**의 절약입니다. 그 기록을 만드는 비용(이번에는 Gemini 약 1만 토큰)은 다른 곳에서 냅니다. 또 무료 Gemini가 "Gemini paused on transient"를 반복하며 기록이 약 7분 늦어졌습니다. 작업을 마치고 바로 새 세션을 열면 방금 한 일이 아직 기억에 없을 수 있다는 뜻입니다.

---

## 뒷정리 — 가짜 정보 3곳 제거

실험용 가짜 정보가 앞으로의 세션에 섞이지 않도록 3곳을 모두 지웠습니다.

1. **레포**: `docs/category-rules.md`를 삭제하고 `git status`가 깨끗한지 확인했습니다.
2. **auto memory**: 복구하다가 `mv` 함정을 만났습니다. 세션 2가 빈 `memory/` 폴더를 새로 만들어둔 상태에서 `mv memory.off memory`를 실행하자, 이름이 바뀌지 않고 `memory/memory.off`로 **안에 중첩 이동**됐습니다. 대상 폴더가 이미 있으면 `mv`는 그 안으로 옮기기 때문입니다. 내용을 확인해 실험 메모 2개만 있는 것을 보고 삭제했습니다.
3. **claude-mem DB**: 워커를 멈추고 정상 종료 표시(`.worker-clean-shutdown`)를 확인한 뒤 백업 폴더로 옮겼습니다.

```bash
mkdir -p ~/claude-mem-test-backup-20261014
cd ~/.claude-mem
mv claude-mem.db claude-mem.db-shm claude-mem.db-wal chroma chroma-sync-state.json state ~/claude-mem-test-backup-20261014/
```

DB 파일 3개는 함께 옮겼습니다. `-wal`은 아직 본 DB에 반영되지 않은 변경을 담은 파일이라 따로 남기면 안 됩니다. `state/`는 대기 중이던 기록이 새 DB로 다시 흘러들어오는 것을 막으려고 옮겼고, 설정·텔레메트리·로그 파일은 유지했습니다. 새 세션에서 "This project has no memory yet"을 확인한 뒤 백업을 삭제했습니다.

---

## 판정: 조건부 유지 (ai-news-monitor 한정, 3주 시험)

**장점**
- 명시적 호출 없이 자동으로 기록됩니다. Memory MCP의 한계를 해결합니다.
- 2층 구조라 평소 컨텍스트 부담이 작습니다.
- 관찰을 decision, sensitive 같은 유형으로 분류합니다.
- provider를 바꿔 구독을 쓰지 않는 구성이 가능합니다.

**단점**
- 설치 함정이 많습니다. 유료 기본 선택, 텔레메트리 동의 생략, 첫 실행 hook 오류, 구독 토큰 기본 사용이 모두 기본값에 숨어 있습니다.
- "자동 기억"은 사실 "자동 목차"입니다.
- 무료 Gemini는 느리고 데이터 사용 조건이 있어 민감한 레포에는 쓸 수 없습니다.
- 내장 auto memory와 역할이 겹칩니다. 이번 실험에서 auto memory는 "미확정, 적지 말 것"까지 한 줄로 매 세션 전달했습니다.

3주 후 판단 기준은 두 가지입니다. `get_observations`가 실제로 쓰였는가, 그리고 재설명 시간이 줄었는가입니다.

---

## 트러블슈팅

| 현상 | 원인 | 해결 |
|---|---|---|
| 설치기가 유료 CMEM Pro를 기본 선택 | 설치기 기본값 | `/plugin` 방식 설치 |
| `bun: command not found` | `/plugin` 방식은 Bun 자동 설치 안 함 | `brew install oven-sh/bun/bun` |
| Gemini 키가 `AQ.` 형식 | 새 형식으로 추정 | models API 호출로 실제 검증 |
| SessionStart `Executable not found in $PATH` | Setup hook과 SessionStart hook의 첫 실행 경합 | 재시작 |
| `--dangerously-skip-permissions`가 계속 붙음 | 셸 히스토리(↑) 재사용 | `claude`만 입력 |
| authMethod에 OAuth 표시 유지 | 표시 의미 불명 | 로그·문서로 확인 + Claude 호출 스킬 미사용 |
| 실험에 auto memory 혼입 | Claude Code 자체 기억 | 폴더 일시 이동으로 변인 통제 |
| 복구한 memory 폴더 안에 memory.off 중첩 | 대상 폴더가 있으면 `mv`는 안으로 이동 | 이름이 아니라 내용까지 `ls`로 확인 |
| `mdfind` 출력에 로그 2줄 | 로케일 로딩 로그 | 검색 결과는 0건으로 판단 |

---

## 실무 활용 시나리오

1. **반복 파이프라인 레포**: 매일 도는 뉴스 수집처럼 세션이 많이 쌓이는 곳에서 작업 이력 타임라인을 자동으로 축적합니다.
2. **장기 프로젝트 복귀**: "지난주에 무엇을 결정했는지"를 목차로 확인하고, 필요한 것만 열람합니다.
3. **팀 AI 도구 도입 정책**: 레포 민감도별 provider 선택, 텔레메트리·fallback 기본값 점검을 도입 체크리스트로 만듭니다.
4. **AI 도구 평가 방법론**: 탐침 실험 + 변인 통제 + 로그 검증을 다른 AI 도구 평가에도 그대로 적용합니다.

---

## 마무리

이번 실습에서는 설치 기본값 속 함정을 먼저 통제하고, 데이터가 어디로 가는지 로그로 확인한 뒤, 탐침 실험으로 "무엇이 실제로 기억되는가"를 검증했습니다. 소개 문구의 "자동 기억"과 실제 동작인 "자동 목차 + 필요 시 열람" 사이의 간극은 직접 실험해 보지 않았다면 알기 어려웠을 것입니다.

다음에는 3주 시험이 끝난 뒤 claude-mem 판정 후기를 정리하고, 프롬프트마다 관련 관찰을 자동으로 붙여주는 `CLAUDE_MEM_SEMANTIC_INJECT` 설정이 "자동 목차"의 한계를 보완하는지 실험해볼 예정입니다. 가이드의 마지막 남은 도구인 Task Observer는 졸업작품 시연 이후에 다루겠습니다.
