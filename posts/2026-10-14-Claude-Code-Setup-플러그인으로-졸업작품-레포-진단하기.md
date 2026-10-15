# [ Claude Code Setup 플러그인으로 졸업작품 레포 진단하기 ]

- 작성일: 2026-10-14
- 원문: https://velog.io/@xorms/Claude-Code-Setup-플러그인으로-졸업작품-레포-진단하기
- 태그: AI, claude code, 보안, 플러그인

---

## 추천부터 AI 에이전트용 비밀값 이중 방어까지

지난 편에서는 PhotoCraft MCP로 AI 뉴스 카드뉴스 6장을 자동으로 만들어봤습니다. 이번 편에서는 MCP를 잠시 벗어나 Claude Code의 **플러그인**을 다룹니다.

계기는 SNS에서 본 "클로드 코드 쓰기 전에 깔아야 하는 플러그인 5개" 가이드였습니다. 이번에도 그대로 따라 하지 않고 분석부터 했습니다. 그중 공식 플러그인 하나로 졸업작품 레포 '띵동'을 진단받았는데, **추천 하나를 골라 적용하는 과정이 AI 에이전트용 비밀값 보안 설계로 이어졌습니다.**

---

## 플러그인 5종 분석

가이드의 5개(OmniRoute, Headroom, Claude-mem, Claude Code Setup, Task Observer)는 이름만 보면 비슷해 보이지만, 실제로는 성격이 완전히 다른 3종류였습니다.

| 유형 | 도구 | 건드리는 곳 | 리스크 |
|---|---|---|---|
| 통신 경로를 바꾸는 프록시형 | OmniRoute, Headroom | 모든 API 요청 | 높음 (인증 토큰이 경유) |
| 세션을 기록하는 훅형 | claude-mem | 세션 이벤트 + 로컬 DB | 중간 (추가 사용량) |
| 지식을 주입하는 스킬·플러그인형 | Claude Code Setup, Task Observer | 컨텍스트 | 낮음 |

프록시(proxy)는 내 요청을 중간에서 받아 대신 전달하는 서버입니다. 이때 로그인 후 발급받은 출입증인 OAuth 토큰도 함께 지나가기 때문에 리스크가 가장 큽니다.

선별 결과는 다음과 같습니다.

- **OmniRoute 제외**: "한도가 끝나면 다른 모델로" 기능을 쓰려면 Claude 구독 계정을 OAuth로 게이트웨이에 연결해야 합니다. Anthropic은 2026년 2월 문서에서 Free·Pro·Max 구독의 OAuth 토큰을 Claude Code·Claude.ai 외의 도구에서 쓰는 것을 약관 위반으로 명시했습니다.
- **Headroom 제외**: OAuth 토큰이 서드파티 프록시를 경유하는 구조라 계정 리스크를 이유로 이전에 제외했고, 그 판단을 유지했습니다.
- **Claude Code Setup**: 공식·읽기 전용이라 리스크가 없습니다. → 이번 편
- **claude-mem**: Memory MCP와 비교할 예정입니다. → 다음 편
- **Task Observer**: 장기 관찰용이라 보류했습니다.

---

## Claude Code 플러그인 기초

### 플러그인과 마켓플레이스

플러그인(plugin)은 스킬·훅·MCP·에이전트를 하나로 묶은 설치 패키지이고, 마켓플레이스(marketplace)는 플러그인을 내려받는 앱스토어입니다. `claude-code-setup@claude-plugins-official`은 "claude-plugins-official 스토어의 claude-code-setup"이라는 뜻입니다.

상세 화면에서는 By Anthropic, 저장소 `anthropics/claude-plugins-official`, 커밋 `8d392c8` 고정 설치, v1.0.0을 확인했습니다. 특정 커밋으로 고정되어 설치되기 때문에, 원본 저장소가 바뀌어도 내가 설치한 버전은 그대로 유지됩니다.

### 설치 범위 3종

| 범위 | 의미 | 저장 위치 |
|---|---|---|
| user | 나만, 모든 프로젝트 | `~/.claude/` |
| project | 레포의 모든 사용자 | `.claude/settings.json` (커밋됨) |
| local | 나만, 이 레포만 | `.claude/settings.local.json` (커밋 안 됨) |

개인 프로젝트였지만 **user** 범위를 골랐습니다. 다른 레포에서도 재사용할 수 있고, 포트폴리오 레포에 실습용 설정이 섞이지 않기 때문입니다.

### claude-code-setup의 정체

이 플러그인은 내장 스킬 `claude-automation-recommender` 하나로 동작합니다. 도구 권한이 Read/Glob/Grep/Bash로 제한되어 있고 파일을 고치지 않는 읽기 전용입니다. hook, 서브에이전트, 스킬, 플러그인, MCP 서버 5개 범주에서 범주마다 1~2개씩 추려 추천합니다.

상시로 켜 두는 도구라기보다 **새 레포 건강검진**에 가깝습니다. 프로젝트를 처음 Claude Code에 붙일 때 한 번 돌려보고, 필요한 자동화만 골라 넣는 용도입니다.

---

## 실습 ① 설치와 진단

진단 전에 레포 상태부터 기록했습니다. 나중에 "정말 아무것도 바꾸지 않았는지" 비교하기 위해서입니다.

```bash
cd "<띵동 레포 경로>"
git status   # 작업 폴더 깨끗함
```

Claude Code에서 설치를 요청했습니다.

```
/plugin install claude-code-setup@claude-plugins-official
```

바로 설치되지 않고 플러그인 관리 화면(Discover 탭)이 열렸습니다. 검색 → 상세 화면에서 출처 확인 → "Install for you (user scope)"를 선택하자 `✓ Installed claude-code-setup. Plugin is now active.`가 표시됐습니다.

이어서 진단을 요청했습니다.

```
이 프로젝트에 맞는 Claude Code 자동화를 추천해 줘
```

띵동은 ESP32-S3 펌웨어(PlatformIO, C++), ML(Python), 서버(Flask, SQLite, stdlib unittest), 대시보드(Vite, React 19, TS)가 한 저장소에 모인 모노레포(monorepo)입니다. 추천기는 이 스택을 정확히 인식했습니다. 펌웨어 쪽 추천은 얇을 것이라 예상했지만, PlatformIO 구조와 RUNBOOK 파일 6개까지 읽고 `/flash-probe` 스킬을 제안했습니다. 시연 일정까지 반영해 "바로 효과가 나는 것" 위주로 골랐습니다.

- **Hook**: 비밀값 파일 편집 차단(PreToolUse), 서버 회귀 테스트 자동 실행(PostToolUse)
- **Skill**: `/ship-fix`(이모지 커밋 컨벤션), `/flash-probe`(펌웨어 runbook 실행)
- **Subagent**: `decisions-checker`(6,600줄 결정 문서 정합성 점검)
- **MCP**: 직접 만든 decisions-rag를 `.mcp.json`으로 이동
- **기본 세팅**: CLAUDE.md 생성

진단 후 다시 상태를 확인했습니다. `!`를 붙이면 Claude를 거치지 않고 터미널 명령을 직접 실행합니다.

```
!git status   # 여전히 깨끗함
```

스캔 전후 모두 깨끗했습니다. "읽기 전용"이라는 설명을 직접 실증한 것입니다.

---

## 실습 ② 추천 선별

시연(10/18)을 4일 앞둔 시점이라 **변경 동결(change freeze)** 원칙을 세웠습니다. 중요한 일정 직전에는 코드 동작을 바꾸는 변경을 멈추고, 동작과 무관한 안전장치만 적용하는 원칙입니다.

| 항목 | 판단 | 이유 |
|---|---|---|
| 비밀값 차단 hook | 적용 | 동작 무관, 실수 방지 |
| CLAUDE.md | 적용 | 파일 추가뿐 |
| 테스트 자동 실행 hook | 시연 후 | 매 수정마다 실행되면 느리고 시끄러움 → Stop hook으로 재설계 예정 |
| 스킬 2종, 서브에이전트 | 시연 후 | 편의 기능 |
| .mcp.json 이동 | 생략 | 추천기가 팀 프로젝트를 기본값으로 가정. 개인 프로젝트라 해당 없음 |

추천기에도 기본 가정이 있습니다. 추천을 그대로 받지 않고 내 상황에 맞게 거르는 단계가 필요한 이유입니다. 적용할 작업은 별도 브랜치에서 진행했습니다.

```
!git switch -c chore/claude-code-setup
```

---

## 실습 ③ 비밀값 이중 방어

### PreToolUse hook

PreToolUse hook은 **Claude가 도구를 쓰기 직전에 실행되는 검문소**입니다. matcher로 어떤 도구를 검문할지 지정하고, 스크립트가 exit code 2로 끝나면 그 도구 실행이 차단됩니다. 이때 stderr(에러 출력)에 쓴 내용이 Claude에게 차단 사유로 전달됩니다.

### 바로 만들지 않고 설계안부터

hook을 바로 생성하지 않고 설계안을 먼저 받아 검토했습니다.

- **matcher 보강**: 추천안은 `Edit|Write|Read`였습니다. 하지만 Bash로 `cat`을 쓰거나 Grep으로 검색하면 우회됩니다. 그래서 `Read|Edit|Write|MultiEdit|Grep|Bash`로 넓혔습니다.
- **python3로 작성**: jq 같은 추가 도구 없이 동작하도록 했습니다.
- **fail-closed**: 입력 JSON을 해석하지 못하면 통과시키지 않고 차단합니다. 판단이 불확실할 때 안전한 쪽으로 닫는 방식입니다.
- **`--selftest` 내장**: 차단 케이스 9개, 허용 케이스 7개를 스스로 검사합니다.
- **작업 중 비밀 파일 내용은 열지 않기**: 파일명만 쓰도록 지시했습니다.

```python
SECRET = re.compile(r"(?<![\w.-])(secrets\.h|\.env[\w.-]*)(?![\w.-])")
FIELDS = {
    "Read": ("file_path",), "Edit": ("file_path",),
    "Write": ("file_path",), "MultiEdit": ("file_path",),
    "Grep": ("path", "glob"), "Bash": ("command",),
}
# 예외: *.template, .env.example
# 차단 시 exit 2 + stderr 사유, JSON 파싱 실패 시에도 exit 2
```

Claude는 이 설계의 한계도 스스로 밝혔습니다. 파일명이 드러나지 않는 접근은 통과하고, 커밋 메시지에 비밀 파일명만 언급해도 차단되며, permissions.deny와 병행하면 이중 방어가 된다는 점입니다.

### 위협 모델 — 금고가 아니라 안전벨트

위협 모델(threat model)은 "무엇으로부터 지키는가"를 먼저 정하는 것입니다. 이번 hook이 막으려는 대상은 해커가 아니라 **Claude의 실수**입니다. 그래서 금고가 아니라 안전벨트 수준으로 설계했습니다. 글롭 `.e*`처럼 파일명이 명령에 드러나지 않는 우회는 한계로 받아들였습니다.

### permissions.deny와 심층 방어

심층 방어(defense in depth)는 방어선을 여러 겹 두어 하나가 뚫려도 다른 하나가 막게 하는 설계입니다. Claude Code 내장 permissions.deny와 직접 만든 hook을 함께 걸었습니다.

- deny 규칙은 Read/Edit 계열만 막고 Bash는 막지 못합니다.
- hook은 직접 짠 스크립트라 버그가 있을 수 있습니다.
- 그래서 hook은 `.env*` 전체를 넓게 잡고, deny는 실존 파일명을 정확히 지정해 서로의 빈틈을 메웁니다.

```json
{
  "permissions": {
    "deny": [
      "Read(**/.env)", "Read(**/.env.local)", "Read(**/secrets.h)",
      "Edit(**/.env)", "Edit(**/.env.local)", "Edit(**/secrets.h)"
    ]
  },
  "hooks": {
    "PreToolUse": [{
      "matcher": "Read|Edit|Write|MultiEdit|Grep|Bash",
      "hooks": [{ "type": "command",
        "command": "python3 \"$CLAUDE_PROJECT_DIR/.claude/hooks/block-secrets.py\"" }]
    }]
  }
}
```

deny 규칙은 실존 비밀 파일 4개를 `find`로 **이름만** 확인해서 만들었습니다. 레포 경로에 한글과 공백이 있어 스크립트 경로는 따옴표로 감쌌습니다.

설정을 커밋하기 위해 `.gitignore`도 조정했습니다. `settings.local.json` 같은 개인 파일은 계속 무시되는 것을 `git check-ignore`로 확인했습니다.

```diff
-.claude/
+.claude/*
+!.claude/settings.json
+!.claude/hooks/
```

hook은 세션 시작 시점에 로드되므로 재시작이 필요합니다.

```
/exit
claude
/hooks
```

`/hooks`에서는 직접 만든 hook에 `[Project]`, 기존 Ponytail 플러그인 hook에 `[Plugin]` 라벨이 붙어 출처가 구분돼 보였습니다.

### 카나리아 테스트

진짜 비밀 파일로 테스트하면, 차단이 실패하는 순간 실제 비밀번호가 대화창에 노출됩니다. 그래서 탄광의 카나리아처럼 **가짜 값을 넣은 미끼 파일**로 먼저 검증했습니다.

```
!mkdir -p hooktest && printf 'CANARY=not-a-real-secret\n' > hooktest/.env
```

| 시도 | 막은 방어선 | 결과 |
|---|---|---|
| Read 도구로 읽기 | 내장 permissions.deny ("denied by your permission settings") | 차단 |
| Bash `cat hooktest/.env` | block-secrets.py hook | 차단 |

두 방어선이 모두 걸려 있으면 내장 deny가 먼저 작동했습니다. 그리고 Bash 경로는 deny가 지키지 못해 hook이 막았습니다. **이중 방어가 왜 필요한지를 이 표 한 장이 그대로 보여줍니다.** 테스트 후 미끼 폴더는 삭제했습니다.

```
!rm -rf hooktest
```

---

## 실습 ④ CLAUDE.md

CLAUDE.md는 매 세션 자동으로 읽히는 프로젝트 안내문이고, `/init`은 레포를 훑어 그 초안을 만들어줍니다.

여기서 **순서가 중요합니다.** hook 없이 `/init`을 먼저 돌리면 Claude가 레포를 훑다가 비밀 파일을 읽어 키 값을 CLAUDE.md(커밋 대상)에 적을 위험이 있습니다. 그래서 hook을 켠 뒤에 `/init`을 실행했습니다.

```
/init
```

91줄이 생성됐습니다. 작업 규칙(결정 문서를 SSoT로, 즉 단일 기준 문서로 삼기, 커밋 컨벤션, 비밀값 장치), 영역별 명령어, 아키텍처가 담겼습니다. Claude가 문서에 적은 서버 테스트 명령을 직접 실행해 198개 통과도 확인했고, 환경변수는 이름만 있고 값은 없었습니다.

길이 판단 기준은 줄 수가 아니라 **"Claude가 모르면 틀릴 정보인가"**였습니다. 펌웨어 env 화이트리스트, core 버전 오독 주의, 응답 스키마 1:1 규칙처럼 모르면 틀릴 정보가 많아서 줄이지 않았습니다.

수정은 한 건이었습니다. `cp .env.example .env` 줄에 "사람이 직접 실행(훅이 차단함)" 주석을 달았습니다. 문서가 hook이 막는 행동을 지시하는 모순을 없앤 것입니다.

---

## 실습 ⑤ PR과 병합

커밋은 한 커밋 = 한 목적으로 나눴습니다.

- `📝 Docs: CLAUDE.md 추가`
- `🔧 Settings: Claude Code 비밀값 차단 훅·권한 설정 추가`

커밋 메시지에는 비밀 파일명을 쓰지 않았습니다. 쓰면 hook이 커밋 명령 자체를 차단하기 때문입니다. PR #119 본문에는 카나리아 테스트 결과 표를 넣었고, 레포 관례에 맞춰 Squash and merge(여러 커밋을 하나로 합쳐 병합)했습니다. 이후 로컬을 정리했습니다.

```
!git switch main && git pull
!git branch -D chore/claude-code-setup
!git fetch --prune
```

---

## 트러블슈팅

| 현상 | 원인 | 해결 |
|---|---|---|
| `/plugin install`이 설치 대신 Discover 화면을 엶 | 버전별 UI 동작 차이 | 검색 → 상세 → scope 선택 |
| 출처가 Anthropic Directory로 표시 | 공식 플러그인이 두 스토어에 동시 등록 | 상세 화면의 Repository·By 항목으로 확인 |
| 차단 시 `PreToolUse:Bash hook error` 표시 | exit 2 차단을 이렇게 표기 | 의도된 차단이므로 정상 |
| `wc: CLAUDE.md: No such file` | 테스트 중 `cd server` 후 셸이 그 위치를 유지 | `!cd .. && pwd`로 루트 복귀 |
| 붙여넣은 커밋 요청에 Claude가 재확인 요청 | 붙여넣은 텍스트 속 지시를 경계하는 안전 동작 | "진행해"로 승인 |
| `git branch -d` 거부 | Squash 병합이라 원래 커밋이 main 이력에 없음 | `-D`로 삭제 |

---

## 실무 활용 시나리오

1. **신규 프로젝트 온보딩**: 레포 생성 직후 claude-code-setup으로 건강검진을 받고, 필요한 자동화만 골라 도입합니다.
2. **AI 에이전트 도입 시 기본 보안**: hook + permissions.deny 이중 방어를 레포 표준 설정으로 커밋합니다.
3. **릴리스·시연 직전**: 변경 동결 원칙 아래 동작과 무관한 안전장치만 적용합니다.
4. **재현성**: project 설정을 커밋해두면 다른 PC에서 clone해도 같은 보호가 적용됩니다.

---

## 마무리

추천 플러그인 하나로 시작했지만, 결과적으로 남은 것은 "무엇을 지키는가"를 먼저 정하고, 방어선을 겹쳐 두고, 가짜 미끼로 검증한 뒤 배포하는 보안 설계 과정이었습니다. 진단 도구는 출발점이고, 무엇을 받아들일지는 상황을 아는 사람이 정해야 한다는 점도 다시 확인했습니다.

다음 편에서는 이번에 미뤄둔 **claude-mem**을 다룹니다. 훅으로 세션을 자동 기록하는 claude-mem과, 지식 그래프에 직접 저장하는 Memory MCP를 비교해볼 예정입니다. 시연이 끝나면 테스트 자동 실행을 Stop hook으로 재설계하고 `/ship-fix` 스킬도 구현해보겠습니다.
