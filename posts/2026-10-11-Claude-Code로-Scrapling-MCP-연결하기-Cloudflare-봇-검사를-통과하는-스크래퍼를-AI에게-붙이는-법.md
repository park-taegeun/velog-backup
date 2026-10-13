# [ Claude Code로 Scrapling MCP 연결 ]

- 작성일: 2026-10-11
- 원문: https://velog.io/@xorms/Claude-Code로-Scrapling-MCP-연결하기-Cloudflare-봇-검사를-통과하는-스크래퍼를-AI에게-붙이는-법
- 태그: Cloudflare, MCP, Scrapling, claudecode, 스크래핑

---

## Cloudflare 봇 검사를 통과하는 스크래퍼를 AI에게 붙이는 법

지난 편에서는 Claude Code의 Nested Agent로 메인 에이전트가 여러 서브에이전트를 병렬로 지휘하는 구조를 실습했습니다. 이번 편에서는 다시 웹 데이터 수집으로 돌아갑니다.

웹 스크래퍼(scraper, 웹 페이지에서 원하는 데이터를 자동으로 가져오는 프로그램)는 보통 두 번 막힙니다. 한 번은 사이트 구조가 바뀌어서, 또 한 번은 봇 검사에 걸려서입니다. **Scrapling MCP를 연결하면 Cloudflare 봇 검사로 막힌 페이지를 자연어 한 줄로 수집하고, 필요한 부분만 잘라서 AI에게 넘길 수 있습니다.**

---

## Scrapling이란 무엇인가

**Scrapling**은 웹 페이지를 가져오고(fetch) 필요한 부분만 뽑아내는(parse) 파이썬 오픈소스 스크래핑 라이브러리입니다. BSD-3 라이선스로 공개되어 있고 GitHub 스타 8만 개 이상을 기록했습니다. 이번 실습은 v0.4.15 기준입니다. MCP 서버가 내장되어 있어 별도 작업 없이 Claude Code에 연결할 수 있습니다.

앞서 말한 두 가지 벽에 Scrapling은 각각 다른 방식으로 대응합니다.

- **사이트 구조 변경**: 셀렉터 규칙이 깨지는 문제 → 적응형 스크래핑(adaptive scraping)으로 대응
- **봇 검사**: Cloudflare "Just a moment..." 화면에서 403 응답 → 이번 편의 핵심

### Cloudflare 봇 검사, 쉽게 이해하기

Cloudflare 봇 검사는 건물 입구의 보안 검색대와 비슷합니다. `curl`은 신분증 없이 문을 두드리는 로봇이라 바로 거절당합니다. Scrapling의 `StealthyFetcher`는 사람처럼 행동하고 검색대 퀴즈까지 스스로 푸는 요원에 가깝습니다.

검색대가 무엇을 보는지 이해하려면 세 가지 용어를 알아야 합니다.

- **브라우저 지문(browser fingerprint)**: 브라우저 버전, 헤더 순서, TLS(웹 통신을 암호화하는 규약) 접속 방식, 화면 크기, 폰트 등의 조합입니다. 접속한 쪽이 진짜 브라우저인지 프로그램인지 판별하는 단서가 됩니다.
- **Turnstile**: Cloudflare가 CAPTCHA 대신 쓰는 챌린지입니다. 백그라운드에서 자동으로 통과되는 유형, 체크박스를 눌러야 하는 interactive 유형 등이 있습니다.
- **리퍼러(referer)**: "어느 페이지에서 왔는지"를 알려주는 요청 정보입니다. Scrapling은 기본적으로 구글 검색에서 들어온 것처럼 설정합니다.

### 3가지 Fetcher와 MCP 도구

Scrapling은 무게가 다른 세 가지 Fetcher를 제공하며, 각각 MCP 도구와 1:1로 대응합니다.

| 파이썬 클래스 | MCP 도구 | 방식 | 언제 쓰나 |
|---|---|---|---|
| Fetcher | `make_request` | HTTP 요청만 (브라우저 X) | 정적 페이지, 가장 빠름 |
| DynamicFetcher | `fetch` | 크로미움 브라우저 실행 | 자바스크립트로 그려지는 페이지 |
| StealthyFetcher | `stealthy_fetch` | 위장 브라우저 + 챌린지 풀이 | 봇 차단 사이트 |

원칙은 단순합니다. **가벼운 것부터 쓰고, 막힐 때만 무거운 것으로 올라갑니다.**

### Playwright / Firecrawl / Scrapling 비교

이 시리즈에서는 이미 웹을 다루는 MCP를 두 개 연결했습니다. 셋은 비슷해 보이지만 역할이 다릅니다.

| 항목 | Playwright MCP | Firecrawl MCP | Scrapling MCP |
|---|---|---|---|
| 실행 위치 | 내 Mac (브라우저) | Firecrawl 클라우드 서버 | 내 Mac (파이썬 + 브라우저) |
| 비용/인증 | 무료, 키 불필요 | API 키 + 크레딧 | 무료 오픈소스, 키 불필요 |
| 주 용도 | UI 조작 (클릭, 로그인) | 대량 수집, map/crawl | 봇 차단 사이트, 정밀 추출 |
| 차별점 | 사람처럼 화면 조작 | 사이트 전체 구조 파악 | CSS 셀렉터로 필요한 부분만 AI에 전달 |
| 약점 | 봇 우회는 주목적 아님 | 외부 서버 경유, 크레딧 소모 | 파이썬 환경 세팅 필요 |

---

## 설치

### 파이썬 버전 선택 — Requires-Python vs Classifiers

PyPI 패키지 정보에는 파이썬 버전 조건이 두 가지 있습니다.

- **Requires-Python (>=3.10)**: pip가 설치를 허용하는 최소 조건
- **Classifiers (3.10~3.13)**: 개발자가 실제로 지원하고 테스트했다고 표시한 버전

macOS 기본 `/usr/bin/python3`는 3.9라 조건 미달이었습니다. Homebrew 기본 `python3`는 3.14라 설치는 되지만 Classifiers 목록 밖입니다. 그래서 이미 설치되어 있던 3.11을 선택했습니다.

```bash
python3 --version                # Python 3.14.4 (Homebrew)
ls /opt/homebrew/bin/python3.1*  # 3.11, 3.14 설치 확인
```

### 가상환경 생성 및 설치

가상환경(virtual environment)은 프로젝트별로 파이썬 패키지를 격리해두는 독립 공간입니다.

```bash
python3.11 -m venv ~/.venvs/scrapling
source ~/.venvs/scrapling/bin/activate   # 프롬프트 앞에 (scrapling) 표시

pip install --upgrade pip
pip install "scrapling[ai]"   # [ai] = MCP 서버까지 포함하는 extras
scrapling install             # 브라우저 설치
```

zsh에서 `pip install scrapling[ai]`처럼 따옴표 없이 입력하면 대괄호를 파일 패턴으로 해석해 `no matches found` 에러가 납니다. 반드시 따옴표로 감싸야 합니다.

브라우저는 `~/Library/Caches/ms-playwright`에 설치되며, Playwright MCP와 같은 캐시 폴더를 공유합니다.

### 설치 로그로 보는 내부 구조

설치 과정에서 함께 설치되는 의존성을 보면 Scrapling이 어떻게 만들어졌는지 보입니다.

| 패키지 | 역할 |
|---|---|
| curl_cffi | HTTP 요청에서 크롬의 TLS 지문을 흉내 냄 → `make_request` |
| playwright | 일반 브라우저 자동화 → `fetch` |
| patchright | 자동화 흔적을 지운 Playwright 개조판 → `stealthy_fetch` |
| browserforge, apify-fingerprint-datapoints | 실제 사용자 통계 기반으로 그럴듯한 브라우저 지문 생성 |
| markdownify | HTML → 마크다운 변환 |
| protego | robots.txt 해석 |
| mcp | MCP 서버 SDK |

세 가지 Fetcher가 각각 다른 엔진 위에서 동작한다는 것을 알 수 있습니다.

---

## 실습 1 — 파이썬으로 Cloudflare 통과 비교

MCP에 연결하기 전에 정말 통과되는지 파이썬으로 먼저 확인했습니다. 대상은 Cloudflare 챌린지 테스트용 데모 페이지입니다.

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://nopecha.com/demo/cloudflare
# 403
```

```python
import time
from scrapling.fetchers import StealthyFetcher

URL = "https://nopecha.com/demo/cloudflare"

for solve in (False, True):
    start = time.time()
    page = StealthyFetcher.fetch(URL, headless=True, solve_cloudflare=solve)
    title = page.css("title::text").get()
    print(f"solve_cloudflare={solve} | {page.status} | {title} | {time.time() - start:.1f}s")
```

| 방법 | 상태 | 페이지 제목 | 시간 |
|---|---|---|---|
| curl | 403 | - | 즉시 |
| StealthyFetcher, `solve_cloudflare=False` | 403 | 잠시만 기다리십시오… | 6.3초 |
| StealthyFetcher, `solve_cloudflare=True` | 200 | NopeCHA - CAPTCHA Demo | 6.6초 |

로그에서 읽을 수 있는 포인트는 다음과 같습니다.

- **위장만으로는 부족합니다.** 브라우저 지문을 위장해도 403이었고, 챌린지를 풀어야 통과했습니다.
- Turnstile **interactive 버전**이 감지되었고 약 3초 만에 해결됐습니다.
- referer가 `https://www.google.com/`으로, 구글 검색 유입처럼 위장되어 있었습니다.
- 제목이 한국어 "잠시만 기다리십시오…"로 나온 것은 브라우저 언어 설정이 반영된 것으로 보입니다.
- 통과 후 **307 임시 리다이렉트**로 `/demo`에 이동해 200을 받았습니다.

참고한 가이드에서는 약 48초가 걸렸는데 실측은 6.6초였습니다. 챌린지 종류, IP, 네트워크 환경 차이로 추정되지만 원인을 단정할 수는 없습니다.

---

## 실습 2 — Claude Code에 MCP 등록

### 왜 절대 경로로 등록해야 하나

Claude Code는 MCP 서버를 직접 실행하지만, 내가 가상환경을 켜두었는지는 알지 못합니다. 그래서 가상환경 안의 실행 파일을 **절대 경로**로 등록해야 합니다.

```bash
which scrapling-mcp
# /Users/xorms/.venvs/scrapling/bin/scrapling-mcp

claude mcp add scrapling --scope user -- "$(which scrapling-mcp)"
claude mcp list
# scrapling: /Users/xorms/.venvs/scrapling/bin/scrapling-mcp - ✔ Connected
```

`$(which scrapling-mcp)`는 명령을 입력하는 순간 절대 경로로 바뀌어 저장됩니다. 따라서 **반드시 가상환경이 켜진 상태에서 등록**해야 합니다. 한 번 등록하면 가상환경을 끈 새 터미널에서도 정상 동작합니다.

참고한 가이드의 명령에는 `--scope user`가 빠져 있어, 그대로 실행하면 현재 폴더에서만 쓰이는 local scope로 등록됩니다. 모든 프로젝트에서 쓰기 위해 user scope로 수정했습니다. `/mcp`에서 User MCPs(`~/.claude.json`) 그룹에 scrapling 도구 13개가 보이면 완료입니다.

---

## 실습 3 — 자연어로 스크래핑

새 터미널(가상환경 비활성 상태)에서 Claude Code를 실행해 두 가지를 요청했습니다.

### stealthy_fetch로 Cloudflare 페이지 수집

```
scrapling의 stealthy_fetch 도구로 https://nopecha.com/demo/cloudflare 를 가져와줘.
solve_cloudflare 옵션을 켜고, 페이지 제목과 본문 핵심만 마크다운으로 보여줘.
```

상태 200, 최종 URL `/demo`, 제목 `NopeCHA - CAPTCHA Demo`로 정상 수집됐습니다.

### make_request + CSS 셀렉터로 토큰 절약

CSS 셀렉터(CSS selector)는 `.price`, `#title`처럼 HTML 요소의 위치를 가리키는 주소입니다.

```
scrapling의 make_request 도구로 https://quotes.toscrape.com 에서
CSS 셀렉터 '.quote .text'에 해당하는 명언 텍스트만 뽑아줘.
```

브라우저를 띄우지 않고 명언 10개만 추출됐습니다. 작가, 태그, 메뉴 같은 나머지 요소는 AI에게 아예 넘어가지 않았습니다. 두 요청을 합쳐 24초가 걸렸습니다.

이 부분이 Scrapling MCP의 진짜 차별점입니다. 다른 스크래핑 MCP는 페이지 전체를 AI에게 넘기고 필요한 필드를 찾게 합니다. Scrapling은 **필요한 요소만 먼저 잘라서 넘기기 때문에 토큰을 훨씬 덜 씁니다.** 여기에 광고·트래커 도메인 약 3,500개 자동 차단, 그리고 숨김 요소·HTML 주석·제로폭 문자 같은 숨겨진 텍스트를 이용한 프롬프트 인젝션(prompt injection, 페이지에 숨겨둔 문장으로 AI를 조종하려는 공격) 방어도 기본으로 적용됩니다.

---

## 트러블슈팅 — AI가 도구 결과를 오해했을 때

이번 실습에서 가장 많이 배운 부분입니다.

### 현상

`stealthy_fetch(solve_cloudflare=true)` 결과를 받은 Claude Code가 이렇게 판단했습니다.

> "Cloudflare 챌린지가 나타나지 않았습니다. solve_cloudflare가 실제로 쓰였는지는 알 수 없습니다."

실습 1에서 직접 로그를 본 입장에서는 챌린지가 분명히 풀린 상황이었습니다. 같은 요청을 두고 AI는 정반대로 해석한 것입니다.

### 원인 — AI는 최종 결과만 본다

```
[MCP 서버 내부]                          [AI에게 전달되는 것]
Turnstile 감지 → 해결 → 307 리다이렉트  →  최종 URL /demo, 200, 본문
(중간 로그는 여기에만 남음)
```

AI는 도구가 돌려준 **최종 결과만** 봅니다. Turnstile 감지, 해결, 307 리다이렉트 같은 중간 과정은 MCP 서버 내부 로그에만 남고 AI에게 전달되지 않습니다. 최종 URL이 `/demo`로 바뀌어 있으니 "처음부터 챌린지가 없었다"고 추론한 것입니다. 주어진 정보 안에서는 그럴듯한 결론이었다는 점이 오히려 주의할 부분입니다.

### 해결 — 대조 실험

같은 URL을 `solve_cloudflare=false`로 한 번 더 요청하게 했습니다.

| 조건 | 상태 | 제목 | 리다이렉트 |
|---|---|---|---|
| `solve_cloudflare=true` | 200 | NopeCHA - CAPTCHA Demo | 있음 (`/demo`) |
| `solve_cloudflare=false` | 403 | 잠시만 기다리십시오… | 없음 |

비교 데이터를 받은 Claude는 직접 true/false 비교표를 만들고, **"최종 URL만 보고 판단해서 챌린지 단계를 놓쳤다"며 스스로 판단을 정정**했습니다.

### 교훈

- **AI의 해석은 AI가 볼 수 있는 정보까지만 정확합니다.**
- 중요한 판단은 대조군(control group, 비교 기준이 되는 실험 조건)을 두고 검증해야 합니다.
- AI에게 비교 데이터를 주면 스스로 오류를 고칠 수 있습니다.

---

## 한계와 윤리적 주의점

- **대응 범위**: Cloudflare Turnstile까지입니다. Akamai, DataDome, Kasada 같은 기업용 방어는 대상이 아닙니다.
- **속도**: 챌린지 해제 옵션을 켜면 일반 요청보다 느립니다. 막히지 않는 사이트는 `make_request`를 쓰는 것이 맞습니다.
- **기술과 허용은 다른 문제입니다.** 기술적으로 넘을 수 있다는 것과 가져가도 된다는 것은 별개입니다. robots.txt, 이용약관, 개인정보, 서버 부하를 반드시 확인해야 합니다.
- 이번 실습은 테스트용 데모 페이지(`nopecha.com/demo`, `quotes.toscrape.com`)로만 진행했습니다.

---

## 마무리

이번 실습으로 Scrapling을 설치하고 Cloudflare 봇 검사 통과를 직접 비교했으며, Claude Code에 MCP로 연결해 자연어로 봇 차단 페이지 수집과 CSS 셀렉터 기반 추출까지 확인했습니다. 이제 웹을 다루는 MCP가 세 개가 됐습니다. 화면 조작은 Playwright, 사이트 전체 수집은 Firecrawl, 봇 차단 사이트와 정밀 추출은 Scrapling이 맡습니다.

실무에서는 AI 뉴스 모니터링 파이프라인에서 Firecrawl로 막히는 공개 소스를 Scrapling으로 보완하거나(약관 확인 전제), 봇 차단이 없는 사이트는 `make_request`로 로컬에서 수집해 Firecrawl 크레딧을 아끼는 방식으로 활용할 수 있습니다.
