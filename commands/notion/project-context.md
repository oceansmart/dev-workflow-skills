# Notion Project Context

## 개요
이 문서는 `/notion:*` 명령어 실행 시 환경 및 프로젝트 컨텍스트를 확인하는 방법을 정의합니다.

**모든 `/notion:*` 명령어는 이 문서의 Phase 0을 필수로 수행해야 합니다.**

---

## ⛔ Phase 0: 환경 및 설정 확인 (BLOCKING - 필수)

### 핵심 규칙
```
Phase 0-1 실패 → MCP 설정 안내 → 완전 중단
Phase 0-1 통과 → Phase 1 실행 가능

※ Notion은 프로젝트별 설정(Phase 0-2)이 필요 없음
   → MCP 연결만 되면 바로 사용 가능
```

---

## Phase 0-1: MCP 연결 확인 (BLOCKING)

### 0-1.1 확인 대상
```
{프로젝트 루트}/.mcp.json
```

### 0-1.2 확인 프로세스
```
1. Read 도구로 .mcp.json 파일 읽기 시도
2. 파일 존재 → "notion" 서버 설정 확인
3. "notion" 설정 존재 → Phase 1로 이동
4. 파일 없음 OR "notion" 설정 없음 → 0-1.3 MCP 설정 안내
```

### 0-1.3 MCP 설정 안내 (설정이 없을 때)

```
⛔ Notion MCP 연결이 설정되지 않았습니다.

/notion:* 명령어를 사용하려면 먼저 Notion MCP 서버 연결이 필요합니다.

📋 필요한 정보:
   • Notion Integration Token
   • (https://www.notion.so/my-integrations 에서 생성)

MCP 설정을 진행할까요? (예/아니오)
```

**사용자 "예" 응답 시:**
```
📝 Notion MCP 설정

다음 정보를 입력해주세요:

1️⃣ Notion Integration Token
   ⚠️ https://www.notion.so/my-integrations 에서 생성

   생성 방법:
   1. 위 링크 접속 → "New integration" 클릭
   2. Name: 원하는 이름 입력 (예: "Claude Code")
   3. Associated workspace: 연결할 워크스페이스 선택
   4. Capabilities: "Read content", "Update content", "Insert content" 활성화
   5. "Submit" 클릭 → Internal Integration Token 복사

   예: secret_abc123...

토큰을 입력해주세요:
```

**입력 수집 후 .mcp.json 생성/업데이트:**
```json
{
  "mcpServers": {
    "notion": {
      "command": "npx",
      "args": ["-y", "@notionhq/notion-mcp-server"],
      "env": {
        "OPENAPI_MCP_HEADERS": "{\"Authorization\": \"Bearer {입력값}\", \"Notion-Version\": \"2022-06-28\"}"
      }
    }
  }
}
```

**⚠️ 기존 .mcp.json이 있는 경우:**
- 기존 설정을 유지하면서 "notion" 서버만 추가
- 다른 MCP 서버 설정(jira, github 등)은 건드리지 않음

**생성 후 메시지:**
```
✅ Notion MCP 설정이 완료되었습니다.

📁 생성/업데이트된 파일: .mcp.json

⚠️ 중요 추가 작업:
   1. Notion에서 연결할 페이지/데이터베이스에 Integration 공유 설정
      - 해당 페이지 열기 → "..." 메뉴 → "Connections" → 생성한 Integration 추가

   2. Claude Code 재시작
      - 현재 세션 종료 (/exit 또는 Ctrl+C)
      - Claude Code 다시 시작
      - /notion:* 명령어 다시 실행

🔒 보안 주의: .mcp.json에 API 토큰이 포함되어 있습니다.
   → .gitignore에 .mcp.json 추가 권장
```

**사용자 "아니오" 응답 시:**
```
❌ 명령어 실행이 취소되었습니다.

/notion:* 명령어를 사용하려면 MCP 설정이 필요합니다.
나중에 설정하려면 다시 /notion:* 명령어를 실행해주세요.
```
→ 명령어 완전 종료

---

## Phase 0 완료 조건

```
✅ Phase 0-1 통과: .mcp.json에 notion 서버 설정 존재

→ Phase 1 (실제 기능) 실행 가능
```

---

## 문제 해결

### MCP 연결 관련 오류

**"Notion MCP 연결이 설정되지 않았습니다":**
→ `.mcp.json` 파일에 notion 서버 설정이 없음
→ Phase 0-1 안내에 따라 MCP 설정 진행

**"MCP 도구를 찾을 수 없습니다":**
→ `.mcp.json`은 있지만 Claude Code 재시작 필요
→ Claude Code 종료 후 다시 시작

**"Notion API 권한 오류" 또는 "페이지를 찾을 수 없습니다":**
→ 해당 페이지에 Integration이 연결되지 않음
→ Notion에서 페이지 열기 → "..." → "Connections" → Integration 추가

**"Notion API 인증 실패":**
→ Integration Token이 만료되었거나 잘못됨
→ https://www.notion.so/my-integrations 에서 토큰 확인/재생성

---

## MCP 서버 정보

### Notion MCP Server
| 항목 | 값 |
|------|-----|
| **Package** | `@notionhq/notion-mcp-server` |
| **Command** | `npx -y @notionhq/notion-mcp-server` |
| **필수 환경변수** | `OPENAPI_MCP_HEADERS` (Authorization, Notion-Version 포함) |
| **토큰 발급** | https://www.notion.so/my-integrations |

### 제공되는 MCP 도구
- `mcp__notion__API-post-search` - 제목 기반 검색
- `mcp__notion__API-retrieve-a-page` - 페이지 조회
- `mcp__notion__API-get-block-children` - 블록(내용) 조회
- `mcp__notion__API-patch-block-children` - 블록 추가
- `mcp__notion__API-query-data-source` - 데이터베이스 쿼리
- `mcp__notion__API-post-page` - 페이지 생성
- `mcp__notion__API-patch-page` - 페이지 수정
- `mcp__notion__API-create-a-comment` - 코멘트 추가
- 기타 다수...

---

## Notion vs Jira 차이점

| 항목 | Notion | Jira |
|------|--------|------|
| **Phase 0-1** | MCP 연결 확인 | MCP 연결 확인 |
| **Phase 0-2** | ❌ 없음 (프로젝트 설정 불필요) | ✅ 있음 (프로젝트 설정 필요) |
| **프로젝트별 설정** | 불필요 | `.claude/config/jira.md` 필요 |
| **이유** | Notion은 워크스페이스 기반으로 동작 | Jira는 프로젝트별 Key, Board ID 등 필요 |

---

*Last Updated: 2025-01-16*
*Version: 1.0.0 - Phase 0-1 (MCP 연결 확인) 추가*
