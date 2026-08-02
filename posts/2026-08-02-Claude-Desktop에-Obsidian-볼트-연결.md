---
title: "[ Claude Desktop에 Obsidian 볼트 연결 ]"
date: 2026-08-02
tags: [ClaudeDesktop, LocalRESTAPI, MCP, obsidian, 트러블슈팅]
velog_url: https://velog.io/@xorms/Claude-Desktop에-Obsidian-볼트-연결
series: AI 생태계 탐험 로그
---

# Claude Desktop에 Obsidian 볼트 연결

지난 편에서는 Claude Code로 Nested Agent 구조를 다루며 서브에이전트 병렬 처리까지 실험했습니다.

이번 편은 조금 결이 다릅니다. 지금까지의 실습이 Claude Code(CLI)에서 MCP를 붙이는 개발자 관점이었다면, 이번에는 Claude Desktop 앱에서 Obsidian 볼트(Vault)를 MCP로 연결하는 소비자용 시나리오를 다룹니다. 같은 MCP지만 클라이언트가 CLI가 아니라 데스크톱 앱이라는 점이 핵심 차이입니다.

그리고 솔직히 말하면, 이번 편은 성공담이라기보다 트러블슈팅 기록에 가깝습니다. 연결 한 번에 붙지 않았고, 원인을 하나씩 배제해 나가는 과정 자체가 이 글의 본체입니다.

## Obsidian × MCP 연결의 구조

먼저 무엇을 연결하는지 정리합니다.

Vault(볼트)는 노트를 담는 로컬 폴더입니다. Obsidian의 노트는 전부 마크다운(.md) 파일로 저장되므로, 파일 시스템 접근만 가능하면 어떤 도구든 바로 읽을 수 있습니다. 이 이식성(portability)이 Obsidian을 MCP와 붙이기 좋은 이유입니다.

연결 구조는 다음과 같습니다.

```
Vault(.md 파일)  ↔  Obsidian + MCP 플러그인(서버)  ↔  Node.js 브리지  ↔  Claude Desktop(클라이언트)  ↔  Claude 모델
```

여기서 반드시 이해해야 할 두 가지가 있습니다.

첫째, MCP 서버는 Obsidian 플러그인 안에서 돕니다. 따라서 Obsidian이 꺼지면 서버도 죽고 연결이 끊깁니다.

둘째, Claude Desktop은 HTTP 방식 MCP를 직접 지원하지 않습니다. 그래서 mcp-remote라는 브리지가 HTTP를 stdio로 변환해 주며, 이 브리지는 Node.js 위에서 실행됩니다.

## 1차 시도 — istefox MCP Connector

처음에는 Obsidian 커뮤니티 플러그인인 MCP Connector(제작자 istefox)로 시작했습니다. .mcpb 확장 파일을 받아 Claude Desktop에 드래그하면 끝나는, 가장 간편한 방식입니다.

설치 자체는 매끄러웠습니다.

- Obsidian에 MCP Connector 설치 + 활성화
- 플러그인 설정 → Quick setup for clients → Download .mcpb
- Claude Desktop → 설정 → 확장 프로그램 → 고급 설정에서 .mcpb 설치
- 파란 커넥터 아이콘 확인, "모든 요구 사항 충족됨" 초록 체크까지 정상

여기까지는 완벽했습니다. 그런데 대화창에서 볼트를 조회하자 문제가 시작됐습니다.

## 트러블슈팅 — 원인을 하나씩 배제하다

증상은 두 가지였습니다. 대화창 상단에 "Could not attach to MCP server Obsidian MCP Connector" 경고가 뜨고, Claude는 볼트를 읽지 못한 채 "사용 가능한 커넥터는 Google Drive, Gmail뿐"이라고 답했습니다.

원인을 하나씩 배제해 나갔습니다.

| 확인 항목 | 명령어 / 방법 | 결과 |
| --- | --- | --- |
| 옵시디언 실행 여부 | 좌하단 볼트 이름 확인 | AI-Vault 정상 |
| 서버 포트 리스닝 | `lsof -i :27200` | Obsidian이 LISTEN 중 ✅ |
| 서버 응답 | `curl -i http://127.0.0.1:27200/` | HTTP 404 (서버 살아있음) ✅ |
| Node.js 설치 | `node --version` | v24.7.0 ✅ |
| Node 설치 위치 | `which node` | /opt/homebrew/bin/node (정상) ✅ |
| Claude 로그 | `mcp-server-....log` | connected successfully ✅ |

여기서 반전이 있었습니다. 로그를 보니 서버는 매번 연결에 성공하고 있었습니다.

```
Server started and connected successfully
Message from client: method="initialize"
... 60초 후 ...
notifications/cancelled
Server transport closed (renderer released port)
```

연결이 붙은 뒤 정확히 60초 후에 스스로 끊기는 패턴이었습니다. 서버 문제가 아니라 Claude Desktop 쪽에서 유휴 타임아웃으로 연결을 끊는 동작이었고, mcp.log에는 에러가 단 한 줄도 없었습니다.

설치·서버·Node·포트·커넥터 등록까지 전부 정상인데 UI만 "연결 실패"를 표시하는 상황. 검색해 보니 MCP 도구 자체는 정상 작동하는데 Claude Desktop UI가 이 에러를 잘못 표시하는 것으로, 도구는 쓸 수 있지만 클라이언트 UI가 에러를 틀리게 보여주는 알려진 이슈가 여럿 있었습니다.

토글 재시작, 앱 완전 재시작(옵시디언 먼저 켜고 Claude 나중에 켜는 순서까지)을 모두 시도했지만 빨간 에러는 사라지지 않았습니다. 이 시점에서 판단을 내렸습니다. 특정 플러그인 버전과 현재 Claude Desktop 빌드의 호환성 문제이며, 여기 매달리는 것보다 검증된 대안으로 갈아타는 편이 낫다는 결론입니다.

## 2차 시도 — Local REST API with MCP로 우회

대안은 레퍼런스가 압도적으로 두꺼운 플러그인이었습니다.

가장 많이 쓰이는 "Local REST API" 플러그인(coddingtonbear, 2,600+ 스타)이 v4.0부터 자체 MCP 서버를 내장했고, v4.1.1에서 이름도 "Local REST API with MCP"로 바뀌었습니다. 별도 서버 설치 없이 플러그인 하나로 끝납니다. 다운로드 수도 630k로, 앞서 실패한 플러그인(17k)과는 검증 차원이 달랐습니다.

이번에는 GUI 드래그가 아니라 설정 파일을 직접 편집하는 방식으로 갔습니다. 앞선 attach 버그를 우회하기 위해서입니다.

1. istefox 흔적 제거 — Claude Desktop 확장 제거 + Obsidian 플러그인 삭제 (27200 포트 반납).
2. Local REST API with MCP 설치 + 활성화.
3. 플러그인 설정에서 HTTP 서버 켜기. 기본은 HTTPS(자체 서명 인증서)라 Claude 연결 시 인증서 문제가 생길 수 있습니다. 평문 HTTP 엔드포인트(http://127.0.0.1:27123/mcp/)를 쓰면 인증서 문제를 통째로 우회할 수 있어서, 설정에서 HTTP 서버를 켜는 것을 권장합니다.
4. `claude_desktop_config.json` 편집. 기존 내용은 그대로 두고 mcpServers만 추가합니다.

```json
{
  "mcpServers": {
    "obsidian": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "http://127.0.0.1:27123/mcp/",
        "--header",
        "Authorization: Bearer <your-api-key>"
      ]
    }
  }
}
```

`<your-api-key>` 자리에는 플러그인 설정의 API Key를 넣습니다. (참고: mcp-remote@latest로 두면 호출할 때마다 최신 버전을 확인하느라 느려집니다. @latest를 빼면 개선됩니다.)

5. Claude Desktop 완전 재시작 후 테스트.

## 마지막 함정 — "빈 볼트"는 버그가 아니었다

설정을 마치고 볼트를 조회하자 Claude는 계속 "볼트가 비어 있다"고 답했습니다. 또 막힌 줄 알았지만, 이번에는 API를 직접 찔러 원인을 특정했습니다.

```bash
curl -s -H "Authorization: Bearer <api-key>" http://127.0.0.1:27123/vault/
# → { "files": [] }
```

빈 배열이 반환됐습니다. 서버도 키도 정상인데 목록이 비어 있었습니다. 원인은 허무할 만큼 단순했습니다. 볼트에 파일이 하나도 없었습니다. 폴더 5개만 만들었을 뿐 .md 파일을 한 개도 넣지 않았고, Local REST API는 빈 폴더를 목록에 포함하지 않았던 것입니다.

테스트 파일 하나를 만들자 결과가 바뀌었습니다.

```bash
# 00-Inbox/test.md 생성 후
curl -s -H "Authorization: Bearer <api-key>" http://127.0.0.1:27123/vault/
# → { "files": ["00-Inbox/"] }
```

폴더가 나타났습니다. 그리고 Claude Desktop에서도 동일하게 확인됐습니다.

> "볼트 루트에는 폴더 하나만 있습니다: 00-Inbox/"

Found tools → Vault read 로그와 함께 Claude가 실제로 볼트를 읽어냈습니다. 연결은 처음부터 정상이었고, "빈 목록"은 버그가 아니라 빈 볼트 때문이었습니다.

## 트러블슈팅 정리

| 증상 | 원인 | 해결 |
| --- | --- | --- |
| Could not attach 반복 | istefox 플러그인 ↔ Claude Desktop 호환성 | Local REST API로 교체 |
| 도구 목록에 Obsidian 없음 | 확장은 등록됐으나 세션에 미노출 | config 직접 편집 방식으로 우회 |
| 호출이 10분씩 지연 | mcp-remote@latest 매번 버전 확인 | @latest 제거 |
| `{"files": []}` 빈 목록 | 볼트에 파일이 없음 (버그 아님) | 파일 생성 후 정상 확인 |

## 마무리

이번 편의 최종 작동 구성은 다음과 같습니다.

- Obsidian AI-Vault + Local REST API with MCP(coddingtonbear)
- HTTP 서버 127.0.0.1:27123
- Claude Desktop `claude_desktop_config.json`에 mcp-remote 브리지 등록

가장 큰 교훈은 두 가지입니다. 첫째, 한 도구에 매달리지 말 것. 설치·서버·환경이 모두 정상인데 연결만 실패한다면 도구 궁합 문제일 수 있고, 그때는 검증된 대안으로 갈아타는 판단이 더 빠릅니다. 둘째, 추측 대신 직접 찔러 볼 것. Claude를 거치지 않고 curl로 API를 직접 호출한 순간, "빈 볼트"라는 진짜 원인이 드러났습니다.

다음 단계에서는 이렇게 연결한 볼트를 실전에 씁니다. 현재 병렬로 운영 중인 프로젝트(공모전, 졸업작품, 인턴 지원 등)의 상태와 다음 할 일을 볼트에 요약해 쌓고, Claude가 이를 가로질러 조회하는 프로젝트 관리 레이어를 구성할 예정입니다.
